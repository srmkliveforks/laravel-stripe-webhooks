**WORK IN PROGRESS, DO NOT USE YET. ETA FOR STABLE RELEASE: END OF OCTOBER 2017**

# Handle Stripe webhooks in a Laravel application

[![Latest Version on Packagist](https://img.shields.io/packagist/v/spatie/laravel-stripe-webhooks.svg?style=flat-square)](https://packagist.org/packages/spatie/laravel-stripe-webhooks)
[![Build Status](https://img.shields.io/travis/spatie/laravel-stripe-webhooks/master.svg?style=flat-square)](https://travis-ci.org/spatie/laravel-stripe-webhooks)
[![StyleCI](https://styleci.io/repos/105920179/shield?branch=master)](https://styleci.io/repos/105920179)
[![SensioLabsInsight](https://img.shields.io/sensiolabs/i/a027b103-772c-4dbc-a2a4-a6ccc07e127f.svg?style=flat-square)](https://insight.sensiolabs.com/projects/a027b103-772c-4dbc-a2a4-a6ccc07e127f)
[![Quality Score](https://img.shields.io/scrutinizer/g/spatie/laravel-stripe-webhooks.svg?style=flat-square)](https://scrutinizer-ci.com/g/spatie/laravel-stripe-webhooks)
[![Total Downloads](https://img.shields.io/packagist/dt/spatie/laravel-stripe-webhooks.svg?style=flat-square)](https://packagist.org/packages/spatie/laravel-stripe-webhooks)

[Stripe](https://stripe.com) can notify your application of events using webhooks. This package can help you handle those webhooks. Out of the box it will verify the stripe signature of all incoming requests. All valid calls will be logged to the database. You can easily define jobs that should be executed when specific events hit your app.

Before using this package we highly recommend reading [the entire documentation on webhooks over at Stripe](https://stripe.com/docs/webhooks).

## Postcardware

You're free to use this package (it's [MIT-licensed](LICENSE.md)), but if it makes it to your production environment we highly appreciate you sending us a postcard from your hometown, mentioning which of our package(s) you are using.

Our address is: Spatie, Samberstraat 69D, 2060 Antwerp, Belgium.

We publish all received postcards [on our company website](https://spatie.be/en/opensource/postcards).

## Installation

You can install the package via composer:

```bash
composer require spatie/laravel-stripe-webhooks
```

The service provider will automatically register itself.

You must publish the config file with:
```bash
php artisan vendor:publish --provider="Spatie\StripeWebhooks\StripeWebhooksServiceProvider" --tag="config"
```

This is the contents of the config file that will be published at `config/stripe-webhooks.php`

```php
return [

    /**
     * Stripe will sign webhooks using a secret. You can find the secret used at the webhook
     * configuration settings: https://dashboard.stripe.com/account/webhooks
     */
    'signing_secret' => '',

    /**
     * Here you can define the job that should be run when a certain webhook hits your .
     * application. The key is name of stripe event type with the `.` replaced by `_`
     *
     * You can find a list of stripe webhook type here:
     * https://stripe.com/docs/api#event_types
     */
    'jobs' => [
        // 'source_chargeable' => \App\Jobs\StripeWebhooks\HandleChargeableSource::class,
        // 'charge_failed' => \App\Jobs\StripeWebhooks\HandleFailedCharge::class,
    ],

    /*
     * The class name of the model to be used.
     */
    'model' => App\Models\StripeWebhookCall::class,
];
```

In the `signing_secret` key of the config file you should add a valid webhook secret.  You can find the secret used at [the webhook configuration settings on the Stripe dashboard](https://dashboard.stripe.com/account/webhooks).

Next, you must publish the migration with:
```bash
php artisan vendor:publish --provider="Spatie\StripeWebhooks\StripeWebhooksServiceProvider" --tag="migrations"
```

After the migration has been published you can create the `stripe_webhook_calls` table by running the migrations:

```bash
php artisan migrate
```

The lasts steps take care of the routing. At [the Stripe dashboard](https://dashboard.stripe.com/account/webhooks) you must configure at what url Stripe webhooks should hit your app. In the routes file of your app you must pass that url to `Route::stripeWebhooks`:

```php
Route::stripeWebhooks('webhook-url-configured-at-the-stripe-dashboard')
```

Behind the scenes this will register a `POST` route to a controller provided by this package. Because Stripe has no way of getting a csrf-token, you must add that route to `except` array of the `VerifyCsrf` middleware .

```php
protected $except = [
    'webhook-url-configured-at-the-stripe-dashboard',
];
```

## Usage

Stripe will send out webhooks for several events. You can find the [full list of events types](https://stripe.com/docs/api#event_types) in the Stripe documentation. If you want to do something when a specific event type comes in you should define a job. Here's an example of such a job.

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;
use Spatie\StripeWebhooks\StripeWebhookCall;

class HandleChargeableSource implements ShouldQueue
{
     use InteractsWithQueue, Queueable, SerializesModels;
    
    /** @var \App\Models\StripeWebhookCall */
    public $webhookCall;

    public function __construct(StripeWebhookCall $webhookCall)
    {
        $this->webhookCall = $webhookCall;
    }

    public function handle()
    {
        // do your work here
        // you can access the payload of the webhook call with `$this->webhookCall->payload`
    }
}
```

We highly recommend that you make this job queueable. By doing that, you can minimize the response time of the webhook requests. This allows you to handle more stripe webhook requests and avoiding timeouts.

After having created your job you must register it at the `jobs` array in the `stripe-webhooks.php` config file. The key should be the name of [the stripe event type](https://stripe.com/docs/api#event_types) where but with the `.` replaced by `_`. The value should be the fully qualified name of the class.

```php
// config/stripe-webhooks.php

'jobs' => [
    'source_chargeable' => \App\Jobs\StripeWebhooks\HandleChargeableSource::class,
],
```

Unless something goes terribly wrong, the response to the webhook call will always be a `200` regardless if you've defined a job that handles the request. Sending a `200` will prevent Stripe from resending the same event over and over again.

### All webhook requests will be logged

All webhook requests with a valid signature will be logged in the `stripe_webhook_calls` table. This happens for all events, regardless of it has a job that handles the event or not. The table has a `payload` column where the entire payload of the incoming webhook is saved.

If something goes wrong during the webhook request the thrown exception will be saved in the `exception` column. In that case the controller will send a `500` instead of `200`.

### Retrying handling a webhook

All incoming webhook requests are written to the database. This is incredibly valueable when something goes wrong handling a webhook call.  Because the job accepted the stored `WebhookCall`, you can easily retry the job after you investigated and fixed the cause of failure.

### Validating requests

Stripe will sign all requests hitting the webhook url of your app. This package will automatically verify if the signature is valid. If it is not, the request was probably not sent be Stripe. The request will not be logged in the `stripe_webhook_calls_table` but a `Spatie\StripeWebhooks\WebhookFailed` exception will be thrown.

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information what has changed recently.

## Testing

```bash
composer test
```

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## Security

If you discover any security related issues, please email freek@spatie.be instead of using the issue tracker.

## Credits

- [Freek Van der Herten](https://github.com/freekmurze)
- [All Contributors](../../contributors)

A big thank you to [Sebastiaan Luca](https://twitter.com/sebastiaanluca) who generously shared his Stripe webhook solution that inspired this package.

## About Spatie

Spatie is a webdesign agency based in Antwerp, Belgium. You'll find an overview of all our open source projects [on our website](https://spatie.be/opensource).

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
