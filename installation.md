<h1 id="doc-title">Installing</h1>

<nav class="toc-nav" markdown="1">

<div class="toc-nav-contents" markdown="1">

<h2 id="table-of-contents">Table of Contents</h2>

1. [Requirements](#requirements)
2. [Installing](#installing)
   1. [Libraries](#libraries)
3. [Server Config](#server-config)
   1. [PHP Built-in Web Server Config](#php-built-in-web-server-config)
   2. [Apache Config](#apache-config)
   3. [Nginx Config](#nginx-config)
   4. [Caddy Config](#caddy-config)
4. [Versioning](#versioning)

</div>

</nav>

<h2 id="requirements">Requirements</h2>

* PHP &ge; 7.4.0

To work around PHP's inconsistencies with where it reads request data from (sometimes superglobals, sometimes `php://input`), Aphiria requires the following setting in either a <a href="http://php.net/manual/en/configuration.file.per-user.php" target="_blank">_.user.ini_</a> or your _php.ini_:

```
enable_post_data_reading = 0
```

This will disable automatically parsing POST data into `$_POST` and uploaded files into `$_FILES`.

> **Note:** If you're developing any non-Aphiria applications on your web server, use <a href="http://php.net/manual/en/configuration.file.per-user.php" target="_blank">_.user.ini_</a> to limit this setting to only your Aphiria application.  Alternatively, you can add `php_value enable_post_data_reading 0` to an _.htaccess_ file or to your _httpd.conf_.

<h2 id="installing">Installing</h2>

Aphiria can be easily downloaded and installed using Composer:

```bash
composer create-project aphiria/app --prefer-dist
```

Be sure to [configure your server](#server-config) to finish the installation.

> **Note:** You can <a href="https://getcomposer.org/download/" target="_blank">download Composer from here</a>.

<h5 id="libraries">Libraries</h5>

Aphiria is broken into various libraries, each of which can be installed individually:

* aphiria/api
* aphiria/application
* aphiria/collections
* aphiria/configuration
* aphiria/console
* aphiria/dependency-injection
* aphiria/exceptions
* aphiria/framework
* aphiria/io
* aphiria/middleware
* aphiria/net
* aphiria/reflection
* aphiria/router
* aphiria/serialization
* aphiria/sessions
* aphiria/validation

<h2 id="server-config">Server Config</h2>

* Aphiria's _tmp_ directory needs to be writable from PHP
* The document root needs to be set to Aphiria's _public_ directory (usually _/var/www/html/public_ or */var/www/html/YOUR_SITE_NAME/public*)

> **Note:** You must set `YOUR_SITE_DOMAIN` and `YOUR_SITE_DIRECTORY` with the appropriate values in the configs below.

<h5 id="php-built-in-web-server-config">PHP Built-in Web Server Config</h5>

To run Aphiria locally, run the following in a terminal:

```bash
php -S localhost:80 -t public localhost_router.php
```
    
This will run PHP's built-in web server. The site will be accessible at http://localhost.

<h5 id="apache-config">Apache Config</h5>

Create a virtual host in your Apache config with the following settings:

```apacheconf
<VirtualHost *:80>
    ServerName YOUR_SITE_DOMAIN
    DocumentRoot YOUR_SITE_DIRECTORY/public

    <Directory YOUR_DOCUMENT_ROOT/public>
        <IfModule mod_rewrite.c>
            RewriteEngine On

            # Handle trailing slashes
            RewriteRule ^(.*)/$ /$1 [L,R=301]

            # Create pretty URLs
            RewriteCond %{REQUEST_FILENAME} !-f
            RewriteRule ^ index.php [L]
        </IfModule>
    </Directory>
</VirtualHost>
```

<h5 id="nginx-config">Nginx Config</h5>

Add the following to your Nginx config:

```nginx
server {
    listen 80;
    server_name YOUR_SITE_DOMAIN;
    root YOUR_SITE_DIRECTORY/public;
    index index.php;
    
    # Handle trailing slashes
    rewrite ^/(.*)/$ /$1 permanent;
    
    # Create pretty URLs
    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }
    
    location ~ \.php$ {
        include                 /etc/nginx/fastcgi_params;
        fastcgi_index           index.php;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_param           SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass            unix:/run/php/php7.0-fpm.sock;
    }
}
```

<h5 id="caddy-config">Caddy Config</h5>

Add the following to your Caddyfile config:

```caddyfile
YOUR_SITE_DOMAIN:80 {
    rewrite {
        r .*
        ext /
        to /index.php?{query}
    }
    fastcgi / 127.0.0.1:9000 php {
        ext .php
        index index.php
    }
}
```

<h2 id="versioning">Versioning</h2>

Aphiria follows semantic versioning 2.0.0.  For more information on semantic versioning, check out its <a href="http://semver.org/" title="Semantic versioning documentation" target="_blank">documentation</a>.
