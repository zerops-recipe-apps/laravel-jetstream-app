<!-- #ZEROPS_REMOVE_START# -->
# Zerops x Laravel Jetstream

[Laravel Jetstream](https://jetstream.laravel.com/introduction.html) is an advanced starter kit by Laravel. [Zerops](https://zerops.io) recipe for Jetstream includes all the advanced functionality — session and cache stored in Redis and files stored in Object Storage, this makes it perfectly suitable for production of any size.

![laravel](https://github.com/zeropsio/recipe-shared-assets/blob/main/covers/svg/cover-laravel.svg)

<br/>

## Deploy on Zerops
You can either click the deploy button to deploy directly on Zerops, or manually copy the [import yaml](https://github.com/zeropsio/recipe-laravel-jetstream/blob/main/zerops-project-import.yml) to the import dialog in the Zerops app.

[![Deploy on Zerops](https://github.com/zeropsio/recipe-shared-assets/blob/main/deploy-button/green/deploy-button.svg)](https://app.zerops.io/recipe/laravel)

<br/>

## Integration Guide
<!-- #ZEROPS_REMOVE_END# -->

> [!TIP]
> One-click deployments use [this repository](https://github.com/zerops-recipe-apps/laravel-jetstream-app) as the deployment source.
> Feel free to explore further by using this repository as a template, or follow the guide below to integrate a similar setup into Zerops.
> For more examples, check out all of our [PHP recipes](https://app.zerops.io/recipes?lf=php).

### 1. Adding `zerops.yaml`
TODO

```yaml
# This example app uses two setups:
# 'prod' for building, deploying, and running the app
# in production or staging environments.
# 'dev' for deploying the source code into a development
# environment with the required toolset.
zerops:
  # Define shared 'base' setup to prevent duplication.
  # Both final setups will use the same subset of environment
  # variables and same Nginx config ('siteConfigPath').
  - setup: base
    run:
      os: ubuntu
      base: php-nginx@8.4
      # This setup uses PHP 8.4, which is compatible with the application's
      # requirement of PHP ^8.2 as specified in composer.json.
      # PHP applications generally require a web server
      # such as Nginx or Apache HTTP server.
      # Read more about it here: https://docs.zerops.io/php/how-to/customize-web-server#customize-nginx-configuration
      siteConfigPath: site.conf.tmpl
      # Most of the environment variables are shared between
      # the two setups, define once and reuse via 'extend' directive.
      envVariables:
        APP_LOCALE: en
        APP_FAKER_LOCALE: en_US
        APP_FALLBACK_LOCALE: en
        APP_MAINTENANCE_DRIVER: file
        APP_MAINTENANCE_STORE: database
        APP_TIMEZONE: UTC
        # Laravel checks the 'Host' header against this value.
        # Feel free to change this value to your own custom domain,
        # after setting up the domain access.
        APP_URL: ${zeropsSubdomain}
        ASSET_URL: ${APP_URL}
        VITE_APP_NAME: ${APP_NAME}

        DB_CONNECTION: pgsql
        DB_HOST: db
        DB_PORT: ${db_port}
        DB_DATABASE: ${db_dbName}
        DB_USERNAME: ${db_user}
        DB_PASSWORD: ${db_password}

        AWS_ACCESS_KEY_ID: ${storage_accessKeyId}
        AWS_REGION: us-east-1
        AWS_BUCKET: ${storage_bucketName}
        AWS_ENDPOINT: ${storage_apiUrl}
        AWS_SECRET_ACCESS_KEY: ${storage_secretAccessKey}
        AWS_URL: ${storage_apiUrl}/${storage_bucketName}
        # Zerops' S3-like object storage uses 'path-style' endpoints,
        # this is required to set with most AWS S3 libraries out there.
        AWS_USE_PATH_STYLE_ENDPOINT: true

        # Zerops automatically collects std out/err
        # of defined commands executions (start, init, build, prepare, ...),
        # and forwards them to syslog.
        # Since PHP is spawned as a worker process by Nginx FastCGI,
        # we have to tell the PHP application to log to the syslog directly,
        # as there is no simple other way to capture std out/err of request execution.
        LOG_CHANNEL: syslog
        LOG_LEVEL: debug
        LOG_STACK: single

        # Configure this to use real SMTP sinks
        # in true production setups. This default configuration
        # expects 'mailpit' to be deployed along the app.
        MAIL_FROM_ADDRESS: hello@example.com
        MAIL_FROM_NAME: ZeropsLaravel
        MAIL_HOST: mailpit
        MAIL_MAILER: smtp
        MAIL_PORT: 1025

        BROADCAST_CONNECTION: redis
        CACHE_PREFIX: cache
        CACHE_STORE: redis
        QUEUE_CONNECTION: redis
        REDIS_CLIENT: phpredis
        REDIS_HOST: redis
        REDIS_PORT: 6379
        SESSION_DRIVER: redis
        SESSION_ENCRYPT: false
        SESSION_LIFETIME: 120
        SESSION_PATH: /

        BCRYPT_ROUNDS: 12
        # Allow running behind reverse proxy (Zerops L7 HTTP balancer service).
        TRUSTED_PROXIES: "*"
        # Specify S3-like object storage as filesystem.
        FILESYSTEM_DISK: s3

  - setup: prod
    extends: base
    # In the 'prod' setup, we have to build and deploy
    # the application, basically get it ready to serve users.
    build:
      # Build in Ubuntu container with PHP 8.4 and Node.js 22 toolsets installed,
      # matching the versions used in the runtime environment.
      os: ubuntu
      base:
        - php@8.4
        - nodejs@22
      # The Laravel Jetstream example app uses Inertia with Vue.js stack
      # (https://jetstream.laravel.com/introduction.html#inertia-vue).
      # Download PHP dependencies via Composer and install Node.js dependencies,
      # then build the Inertia frontend assets including SSR support via npm.
      buildCommands:
        - composer install --optimize-autoloader --no-dev
        - npm install
        - npm run build
      # Deploy all source, fetched and built application files.
      deployFiles: ./
      # Cache dependencies for faster incremental builds.
      # For more information, please visit: https://docs.zerops.io/features/build-cache
      cache:
        - vendor
        - composer.lock
        - node_modules
        - package-lock.json
    # Showing how to define simple readiness check,
    # when new application version is being deployed.
    # If this check fails, the being-deployed version
    # is probably not functional and thus deploy is aborted,
    # leaving the previous working version running.
    # More about it here: https://docs.zerops.io/zerops-yaml/specification#readinesscheck-
    deploy:
      readinessCheck:
        httpGet:
          port: 80
          path: /up
    run:
      # Turn off debug and set other setup-specific environment variables.
      envVariables:
        APP_NAME: ZeropsLaravelJetstreamProd
        APP_DEBUG: false
        APP_ENV: production
      # Run these pre-heat and migration commands,
      # right after creating every runtime container
      # of the given application version (associated with this 'zerops.yaml').
      initCommands:
        - php artisan view:cache
        - php artisan config:cache
        - php artisan route:cache
        - php artisan migrate --isolated --force
        - php artisan optimize
      # Zerops also supports continuous health-checks
      # in already running deployed runtime containers.
      # More about it here: https://docs.zerops.io/zerops-yaml/specification#healthcheck-
      healthCheck:
        httpGet:
          port: 80
          path: /up

  - setup: dev
    extends: base
    # In this development setup used for Remote Development environments
    # or by AI agents, we deploy only the application source code
    # to enable further development and testing.
    build:
      base: ubuntu@latest
      deployFiles: ./
    run:
      # Since runtime containers cannot combine multiple base images,
      # install Node.js manually to support Inertia and Vue.js development.
      prepareCommands:
        - curl -sL https://deb.nodesource.com/setup_22.x -o /tmp/node_setup.sh
        - sudo bash /tmp/node_setup.sh
        - sudo apt-get install nodejs
        - node -v
      # Turn on debug and set other setup-specific environment variables.
      envVariables:
        APP_NAME: ZeropsLaravelJetstreamDev
        APP_DEBUG: true
        APP_ENV: development
      # With only source code deployed, we run these extra commands
      # after container creation to provide a ready-to-dive-in experience.
      initCommands:
        - zsc scale ram +0.5GB 10m
        - composer install
        - npm install
```

### 2. TODO
If you want to modify your existing Laravel/Jetstream app to efficiently run on Zerops, these are the general steps we took:

- Add [zerops.yml](https://github.com/zeropsio/recipe-laravel-jetstream/blob/main/zerops.yml) to your repository, our example includes idempotent migrations, caching, and optimized build process
- Add [league/flysystem-aws-s3-v3](https://github.com/zeropsio/recipe-laravel-jetstream/blob/main/composer.json#L14) to your composer.json to support Object Storage file system
- Setup [Jetstream config](https://github.com/zeropsio/recipe-laravel-jetstream/blob/main/config/jetstream.php#L79) to use object storage for file system
- Utilize Zerops [environment variables](https://github.com/zeropsio/recipe-laravel-jetstream/blob/main/zerops.yml#L25-L75) and [secrets](https://github.com/zeropsio/recipe-laravel-jetstream/blob/main/zerops-project-import.yml#L12-L16) to setup S3 for file system, Redis for cache and sessions, and trusted proxies to work with reverse proxy load balancer

<!-- #ZEROPS_REMOVE_START# -->
## Understand Zerops Core Concepts
If you want to try integrating Zerops from scratch on a new Laravel project, check our [step-by-step tutorial](https://docs.zerops.io/frameworks/laravel/introduction) which demonstrates how to use Zerops effectively with Laravel.

<br/>

## Recipe features

- Laravel + Inertia.js running on a load balanced **Zerops PHP + Nginx** service
- Zerops **PostgreSQL 16** service as database
- Zerops KeyDB (**Redis**) service for session and cache
- Zerops **Object Storage** (S3 compatible) service as file system
- Proper setup for Laravel **cache**, **optimization**, and **database migrations**
- Logs set up to use **syslog** and accessible through Zerops GUI
- Utilization of Zerops built-in **environment variables** system
- [Mailpit](https://github.com/axllent/mailpit) as **SMTP mock server**

<br/>

## Production vs. development

Base of the recipe is ready for production, the difference comes down to:

- Use highly available version of the PostgreSQL database (change `mode` from `NON_HA` to `HA` in recipe YAML, `db` service section)
- Use at least two containers for Jetstream service to achieve high reliability and resilience (add `minContainers: 2` in recipe YAML, `app` service section)
- Use production-ready third-party SMTP server instead of Mailpit (change `MAIL_` secret variables in recipe YAML `app` service)

<br/>

## Changes made over the default installation

If you want to modify your existing Laravel/Jetstream app to efficiently run on Zerops, these are the general steps we took:

- Add [zerops.yml](https://github.com/zeropsio/recipe-laravel-jetstream/blob/main/zerops.yml) to your repository, our example includes idempotent migrations, caching, and optimized build process
- Add [league/flysystem-aws-s3-v3](https://github.com/zeropsio/recipe-laravel-jetstream/blob/main/composer.json#L14) to your composer.json to support Object Storage file system
- Setup [Jetstream config](https://github.com/zeropsio/recipe-laravel-jetstream/blob/main/config/jetstream.php#L79) to use object storage for file system
- Utilize Zerops [environment variables](https://github.com/zeropsio/recipe-laravel-jetstream/blob/main/zerops.yml#L25-L75) and [secrets](https://github.com/zeropsio/recipe-laravel-jetstream/blob/main/zerops-project-import.yml#L12-L16) to setup S3 for file system, Redis for cache and sessions, and trusted proxies to work with reverse proxy load balancer
- In case of true production setup, setup `MAIL_` environments to match your SMTP provider or server.

<br/>
<br/>

Need help setting your project up? Join [Zerops Discord community](https://discord.com/invite/WDvCZ54).
<!-- #ZEROPS_REMOVE_END# -->
