# CKFinder 4 Package for Laravel 9+

This repository contains the CKFinder 4 Package for Laravel 9+.
This repository was forked from https://github.com/ckfinder/ckfinder-laravel-package
Purpose: Fix S3 Upload using IAM role from EC2

<h3 align="center"><a href="https://ckeditor.com/docs/ckfinder/demo/ckfinder3/samples/widget.html"><img src="https://user-images.githubusercontent.com/803299/42693315-18717aae-86af-11e8-863a-74070edb3912.png"></a></h3>

## Installation

1. Add a Composer dependency and install the package.

    ```bash
    namdp06/ckfinder-4.0-fix-s3-iam-upload
    ```

2. Publish the CKFinder connector configuration and assets.

    ```bash
    php artisan vendor:publish --tag=ckfinder-assets --tag=ckfinder-config
    ```

    This will publish CKFinder assets to `public/js/ckfinder`, and the CKFinder connector configuration to `config/ckfinder.php`.
    
    You can also publish the views used by this package in case you need custom route names, different assets location, file browser customization etc.
    
    ```bash
    php artisan vendor:publish --tag=ckfinder-views
    ```
    
    Finally, you can publish package's configuration, assets and views using only one command.
    
    ```bash
    php artisan vendor:publish --tag=ckfinder
    ```

3. Create a directory for CKFinder files and allow for write access to it. By default CKFinder expects the files to be placed in `public/userfiles` (this can be altered in the configuration).

    ```bash
    mkdir -m 777 public/userfiles
    ```

    **NOTE:** Since usually setting permissions to `0777` is insecure, it is advisable to change the group ownership of the directory to the same user as Apache and add group write permissions instead. Please contact your system administrator in case of any doubts.

4. CKFinder by default uses a CSRF protection mechanism based on [double submit cookies](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html#double-submit-cookie). On some configurations it may be required to configure Laravel not to encrypt the cookie set by CKFinder.

   To do that, please add the cookie name `ckCsrfToken` to the `$except` property of `EncryptCookies` middleware:

   ```php
   // app/Http/Middleware/EncryptCookies.php

   namespace App\Http\Middleware;

   use Illuminate\Cookie\Middleware\EncryptCookies as Middleware;

   class EncryptCookies extends Middleware
   {
       /**
        * The names of the cookies that should not be encrypted.
        *
        * @var array
        */
       protected $except = [
           'ckCsrfToken',
           // ...
       ];
   }
   ```

   You should also disable Laravel's CSRF protection for CKFinder's path, as CKFinder uses its own CSRF protection mechanism. This can be done by adding `ckfinder/*` pattern to the `$except` property of `VerifyCsrfToken` middleware:
   (app/Http/Middleware/VerifyCsrfToken.php)

    ```php
    // app/Http/Middleware/VerifyCsrfToken.php

    namespace App\Http\Middleware;

    use Illuminate\Foundation\Http\Middleware\VerifyCsrfToken as Middleware;

    class VerifyCsrfToken extends Middleware
    {
        /**
         * The URIs that should be excluded from CSRF verification.
         *
         * @var array
         */
        protected $except = [
            'ckfinder/*',
            // ...
        ];
    }
    ```

At this point you should see the connector JSON response after navigating to the `<APP BASE URL>/ckfinder/connector?command=Init` address.
Authentication for CKFinder is not configured yet, so you will see an error response saying that CKFinder is not enabled.

## Configuring Authentication

CKFinder connector authentication is handled by [middleware](https://laravel.com/docs/5.8/middleware) class or alias. To create the custom middleware class, use the artisan command:

```bash
php artisan make:middleware CustomCKFinderAuth
```

The new middleware class will appear in `app/Http/Middleware/CustomCKFinderAuth.php`. Change the `authentication` option in `config/ckfinder.php`:

```php
$config['authentication'] = '\App\Http\Middleware\CustomCKFinderAuth';
```

The `handle` method in `CustomCKFinderAuth` class allows to authenticate CKFinder users. A basic implementation that returns `true` from the `authentication` callable (which is obviously **not secure**) can look like below:

```php
public function handle($request, Closure $next)
{
    config(['ckfinder.authentication' => function() {
        return true;
    }]);
    return $next($request);
}
```

Please have a look at the [CKFinder for PHP connector documentation](https://ckeditor.com/docs/ckfinder/ckfinder3-php/configuration.html#configuration_options_authentication) to find out
more about this option.

**Note**:
Alternatively, you can set the configuration option `$config['loadRoutes'] = false;` in `config/ckfinder.php`. Then you copy the routes from `vendor/ckfinder/ckfinder-laravel-package/src/routes.php` to your application routes such as ```routes/web.php``` to protect them with your Laravel auth middleware. 

```php
Route::any('/ckfinder/connector', '\CKSource\CKFinderBridge\Controller\CKFinderController@requestAction')
    ->name('ckfinder_connector');

Route::any('/ckfinder/browser', '\CKSource\CKFinderBridge\Controller\CKFinderController@browserAction')
    ->name('ckfinder_browser');
```

## Configuration Options

The CKFinder connector configuration is taken from the `config/ckfinder.php` file.

To find out more about possible connector configuration options please refer to the [CKFinder for PHP connector documentation](https://ckeditor.com/docs/ckfinder/ckfinder3-php/configuration.html).

## Usage

The package code contains a couple of usage examples that you may find useful. To enable them, uncomment the `ckfinder_examples`
route in `vendor/ckfinder/ckfinder-laravel-package/src/routes.php`:

```php
// vendor/ckfinder/ckfinder-laravel-package/src/routes.php

Route::any('/ckfinder/examples/{example?}', 'CKSource\CKFinderBridge\Controller\CKFinderController@examplesAction')
    ->name('ckfinder_examples');
```

After that you can navigate to the `<APP BASE URL>/ckfinder/examples` path and have a look at the list of available examples.
To find out about the code behind them, check the `views/samples` directory in the package (`vendor/ckfinder/ckfinder-laravel-package/views/samples/`).

### Including the Main CKFinder JavaScript File in Templates

To be able to use CKFinder on a web page you have to include the main CKFinder JavaScript file.
The preferred way to do that is to include the CKFinder setup template, as shown below:

```blade
@include('ckfinder::setup')
```

The included template renders the required `script` tags and configures a valid connector path.

```html
<script type="text/javascript" src="/js/ckfinder/ckfinder.js"></script>
<script>CKFinder.config( { connectorPath: '/ckfinder/connector' } );</script>
```

---

## Useful Links

 * [CKFinder 3 usage examples](https://ckeditor.com/docs/ckfinder/demo/ckfinder3/samples/widget.html)
 * [CKFinder 3 for PHP connector documentation](https://ckeditor.com/docs/ckfinder/ckfinder3-php/)
 * [CKFinder 3 Developer's Guide](https://ckeditor.com/docs/ckfinder/ckfinder3/)
 * [CKFinder 3 issue tracker](https://github.com/ckfinder/ckfinder)
