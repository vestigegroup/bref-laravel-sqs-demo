**This is the documentation for https://github.com/brefphp/laravel-bridge**

This repository also includes a full example on how to use SQS with Laravel Queues on AWS Lambda. You can clone this repository and deploy it via `serverless deploy`. Read [Bref installation](https://bref.sh/docs/installation.html) and [Deploying with Bref](https://bref.sh/docs/deploy.html) if this is your first time.

---

Bridge to use Laravel on AWS Lambda with Bref.

**These instructions apply to Laravel 7.**

## Installation

```bash
composer require bref/laravel-bridge
```

## Laravel Queues with SQS

This package lets you process jobs from SQS queues by integrating with Laravel Queues and its job system.

For example, given [a `ProcessPodcast` job](https://laravel.com/docs/7.x/queues#class-structure):

```php
<?php declare(strict_types=1);

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessPodcast implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /** @var int */
    private $podcastId;

    public function __construct(int $podcastId)
    {
        $this->podcastId = $podcastId;
    }

    public function handle(): void
    {
        // process the job
    }
}
```

We can dispatch this job to SQS [just like any Laravel job](https://laravel.com/docs/7.x/queues#dispatching-jobs):

```php
ProcessPodcast::dispatch($podcastId);
```

The job will be pushed to SQS. Now, instead of running the `php artisan queue:work` command, SQS will directly trigger our **handler** on AWS Lambda to process our job immediately.

## Setup

### SQS

To create the SQS queue (and the permissions for your Lambda to read/write to it), you can either do that manually, or use `serverless.yml`.

Here is a complete example with `serverless.yml` that creates the queue, as well as sets up the permissions:

```yaml
provider:
    ...
    environment:
        APP_ENV: production
        SQS_PREFIX:
            Fn::Join: ['', [ 'https://sqs.us-east-1.amazonaws.com/', { Ref: AWS::AccountId } ]]
        SQS_QUEUE:
            Ref: AlertQueue
    iamRoleStatements:
        # Allows our code to interact with SQS
        -   Effect: Allow
            Action: [sqs:SendMessage, sqs:DeleteMessage]
            Resource:
                Fn::GetAtt: [ AlertQueue, Arn ]

functions:

    website:
        handler: public/index.php
        timeout: 28 # in seconds (API Gateway has a timeout of 29 seconds)
        layers:
            - ${bref:layer.php-73-fpm}
        events:
            -   http: 'ANY /'
            -   http: 'ANY /{proxy+}'

    worker:
        handler: worker.php
        layers:
            - ${bref:layer.php-73}
        events:
            -   sqs:
                    arn:
                        Fn::GetAtt: [ AlertQueue, Arn ]
                    # Only 1 item at a time to simplify error handling
                    batchSize: 1

resources:
    Resources:

        AlertQueue:
            Type: AWS::SQS::Queue
            Properties:
                RedrivePolicy:
                    maxReceiveCount: 3 # jobs will be retried up to 3 times
                    # Failed jobs (after the retries) will be moved to the other queue for storage
                    deadLetterTargetArn:
                        Fn::GetAtt: [ DeadLetterQueue, Arn ]

        # Failed jobs will go into that SQS queue to be stored, until a developer looks at these errors
        DeadLetterQueue:
            Type: AWS::SQS::Queue
            Properties:
                MessageRetentionPeriod: 1209600 # maximum retention: 14 days
```

### Laravel

First, you need to configure [Laravel Queues](https://laravel.com/docs/7.x/queues) to use the SQS queue.

You can achieve this by setting the `QUEUE_CONNECTION` environment variable to `sqs` and configuring the rest:

```dotenv
# .env
QUEUE_CONNECTION=sqs
SQS_PREFIX=https://sqs.us-east-1.amazonaws.com/your-account-id
SQS_QUEUE=my_sqs_queue
AWS_DEFAULT_REGION=us-east-1
```

Note that on AWS Lambda, you do not need to create `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` variables: these access keys are created automatically by Lambda and available through those variables. There is, however, one thing missing: the `AWS_SESSION_TOKEN` variable is not taken into account by Laravel by default (comment on [this issue](https://github.com/laravel/laravel/pull/5138#issuecomment-624025825) if you want this fixed). In the meantime, edit `config/queue.php` to add this line:

```diff
        'sqs' => [
            'driver' => 'sqs',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
+           'token' => env('AWS_SESSION_TOKEN'),
            'prefix' => env('SQS_PREFIX', 'https://sqs.us-east-1.amazonaws.com/your-account-id'),
```

Next, create a `handler.php` file. This is the file that will handle SQS events in AWS Lambda:

```php
<?php declare(strict_types=1);

use Bref\LaravelBridge\Queue\LaravelSqsHandler;
use Illuminate\Foundation\Application;

require __DIR__ . '/vendor/autoload.php';
/** @var Application $app */
$app = require __DIR__ . '/bootstrap/app.php';

$kernel = $app->make(Illuminate\Contracts\Console\Kernel::class);
$kernel->bootstrap();

return $app->makeWith(LaravelSqsHandler::class, [
    'connection' => 'sqs',
    'queue' => getenv('SQS_QUEUE'),
]);
```

You may need to adjust the `connection` and `queue` options above if you customized the configuration in `config/queue.php`. If you are unsure, have a look [at the official Laravel documentation about connections and queues](https://laravel.com/docs/7.x/queues#connections-vs-queues).

We can now configure our handler in `serverless.yml`:

```yaml
functions:
    worker:
        handler: handler.php
        timeout: 20 # in seconds
        reservedConcurrency: 5 # max. 5 messages processed in parallel
        layers:
            - ${bref:layer.php-74}
        events:
            - sqs:
                arn: arn:aws:sqs:us-east-1:1234567890:my_sqs_queue
                # Only 1 item at a time to simplify error handling
                batchSize: 1
```

That's it! Anytime a job is pushed to the `my_sqs_queue`, SQS will invoke `handler.php` and our job will be executed.

### Differences and limitations

The SQS + Lambda integration already has a retry mechanism (with a dead letter queue). This is why those mechanisms from Laravel are not used at all. These should instead be configured on SQS (by default, jobs are retried in a loop for several days).

For those familiar with Lambda, you may know that batch processing implies that any failed job will mark all the other jobs of the batch as "failed". However, Laravel manually marks successful jobs as "completed" (i.e. those are properly deleted from SQS).
