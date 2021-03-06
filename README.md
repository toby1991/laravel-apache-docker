# Laravel Base Image 

This image is a base imagem for Laravel application served by Apache with mod_php running **php 7**.

This image is a lightweight laravel app ready production image.

## Stack

- Apache 2 with PHP mod
- PHP 7
- Node 6.6.2 *(Used only to run frontend tasks. Removed from the final image)*


See `How to use this image` section for more details.

![logo PHP](php-logo.png) ![logo Laravel](laravel-logo.png) ![Apache](httpd-logo.png)

[PHP][1]
[Laravel][2]
[Apache][3]

# Supported tags and respective `Dockerfile` links

* [`5-onbuild` (*5-onbuild/Dockerfile*)](5-onbuild/Dockerfile)

# How to use this image

## Create a `Dockerfile` in your Laravel project:

```dockerfile
FROM brunogasparin/laravel-apache:5-onbuild
```

Put this file in the root of your app, next to the `composer.json`.

This image includes the minimum packages for a Laravel project to work. It do not include database
libraries or memcache libraries, for example. It's up to you to install these libraries. For
example, if you are going to use PostgreSQL as database, you will need to install the libraries
and binaries to communicate to PostgreSQL as well as the postgres php extension.
You can do it in your `Dockerfile`:

```dockerfile
FROM brunogasparin/laravel-apache:5-onbuild

# Install postgres libraries and headers for C language
RUN apt-get update && apt-get install -y \
        libpq-dev \
        && apt-get clean \
        && rm -rf /var/lib/apt/lists/*

# Install postgres php extension
RUN docker-php-ext-install \
        pdo_pgsql \
        pgsql
```

This image includes multiple `ONBUILD` triggers which should cover most Laravel applications.
The build will:

* `ONBUILD COPY composer.json composer.lock artisan /var/www/html/`
* `ONBUILD COPY database /var/www/html/database/`
* `ONBUILD RUN composer install --prefer-dist --optimize-autoloader --no-scripts --no-dev --profile --ignore-platform-reqs -vvv`
* `ONBUILD COPY package.json /var/www/html/`
* `ONBUILD RUN npm install`
* `ONBUILD COPY . /var/www/html`
* `ONBUILD RUN composer run-script post-install-cmd`
* `ONBUILD RUN php artisan clear-compiled`
* `ONBUILD RUN php artisan optimize`
* `ONBUILD RUN php artisan config:cache`
* `ONBUILD RUN chgrp -R www-data storage /var/www/html/bootstrap/cache`
* `ONBUILD RUN chmod -R ug+rwx storage /var/www/html/bootstrap/cache`
* `ONBUILD RUN chgrp -R www-data storage /var/www/html/storage`
* `ONBUILD RUN chmod -R ug+rwx storage /var/www/html/storage `
* `ONBUILD VOLUME /var/www/html/storage`
* `ONBUILD RUN gulp --production`
* `ONBUILD RUN rm -R /var/www/html/node_modules `
* `ONBUILD RUN rm -Rf tests/`
* `ONBUILD RUN rm /usr/local/bin/node \`
  `  && rm /usr/local/bin/npm \`
  `  && rm /usr/local/bin/gulp`
* `ONBUILD RUN apt-get remove -y \`
  `  xz-utils \`
  `  && apt-get clean \`
  `  && rm -rf /var/lib/apt/lists/*l`

Note that the images ignores the composer scripts when running `composer install` command, but run the `post-install-cmd` scripts.

This image also supposes you install your node dependecies into a folder named `node_modules`  inside your application root directory (the default node behavior). 

Also, if your application has custom a composer script, you should added into you Dockerfile:
	
```dockerfile
FROM brunogasparin/laravel-apache:5-onbuild
# ...
RUN composer run-script my-custom-script`
```

The build also sets the default command to `apache2-foreground` to start apache2 service.

You can now build and run the Docker image:

```console
$ docker build -t my-laravel-app .
$ docker run --name some-laravel-app -d my-laravel-app
```

You can test it by visiting `http://container-ip` in a browser or, if you need access outside
the daemon's host, on port 8080:

    $ docker run --name some-laravel-app -p 8080:80 -d my-laravel-app

You can then go to `http://localhost:8080` or `http://host-ip:8080` in a browser.

## Ready for Production 

When you build an image using this base image, the result can image can be used for
your production container:

    $ docker build -t my-production-laravel-app-image .
    $ docker run --name my-producton-container -p 8080:80 -d my-production-laravel-app-image

This cointainer will have all the an laravel application compiled for a production environment.

This is done because during the image build process, :

   - All php composer dependencies  (`gulpfile.json`) were installed
   - All frontend tasks (`gulpfile.js`) was executed for production environment  

So the resulting image will be prepared to be runned into production.

| **Note:** Frontend issues are managed using **6.2.2 node version**.

## But how can I use this image for Development

For development, you`ll probably want to mount your source code into the container:

    $ docker run --name my-producton-container -p 8080:80 -d my-production-laravel-app-image

Using this configuration, you may need to run composer and gulpfile tasks manually.

### How to install more PHP extensions

This images extends the [php:7-apache Oficial Image][3], so it provides two convenient scripts 
named `docker-php-ext-configure` and `docker-php-ext-install`, 
you can use them to easily install PHP extension. See [PHP Oficial Image Repository][3] for 
more details.

### Apache

Apache is configured to provide `Authorization header` to the request.

Apache is also configured to disable .htaccess support on the application directory
to increase its performance.

## PHP

PHP is configured to run optimized (for this image to be used in production). You can change php
configuration adding a custom `php.ini` configuration. `COPY` it into `/usr/local/etc/php` by
adding one more line to your Dockerfile:

    FROM brunogasparin/laravel-apache:5-onbuild
    COPY config/php.ini /usr/local/etc/php

## Configuring Laravel

Laravel is configured using environment variable. So to configure your Laravel application, you can use
docker:

```console
$ docker run --name some-laravel-app -e "APP_DEBUG=false" -e "DB_CONNECTION=pgsql" -p 8080:80 -d my-laravel-app
```

## Development

When using this image for development, its a good practice to mount the application source directory
from your host into the docker container. When you do that, you may probably overwrite the
vendors packages.

If it's yours first time, you will probably need to install the application vendors again:

```console
$ docker exec -u Laravel my-laravel-app composer install --prefer-dist
```

### How to configure PHP for development

This image comes with php-fpm configured to be used in production environment, containing settings which hold 
security, performance and best practices. To customize the php behavior for development environment, `COPY`
 a custom `php.ini` configuration into `/usr/local/etc/php` (you can use the `$PHP_INI_DIR` environment variable) 
 by adding one more line to the Dockerfile above and running the same commands to build and run:

```dockerfile
FROM brunogasparin/laravel-apache:5-onbuild
COPY config/php.ini $PHP_INI_DIR
```

Where `config/` contains your `php.ini` file.

# Useful Links

- [PHP Oficial Image Repository][3]

[1]: http://http://php.net/
[2]: https://laravel.com
[3]: http://httpd.apache.org
