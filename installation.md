<h1 id="doc-title">Installing</h1>

<nav class="toc-nav">

<div class="toc-nav-contents">

<h2 id="table-of-contents">Table of Contents</h2>

<ol>
<li><a href="#requirements">Requirements</a></li>
<li><a href="#installing">Installing</a><ol>
<li><a href="#libraries">Libraries</a></li>
</ol>
</li>
<li><a href="#server-config">Server Config</a><ol>
<li><a href="#php-built-in-web-server-config">PHP Built-in Web Server Config</a></li>
<li><a href="#apache-config">Apache Config</a></li>
<li><a href="#nginx-config">Nginx Config</a></li>
<li><a href="#caddy-config">Caddy Config</a></li>
</ol>
</li>
<li><a href="#versioning">Versioning</a></li>
</ol>

</div>

</nav>

<h2 id="requirements">Requirements</h2>

* PHP &ge; 8.4.0

To work around PHP's inconsistencies with where it reads request data from (sometimes superglobals, sometimes `php://input`), Aphiria requires the following setting in either a <a href="http://php.net/manual/en/configuration.file.per-user.php" target="_blank">_.user.ini_</a> or your _php.ini_:

```
enable_post_data_reading = 0
```

This will disable automatically parsing POST data into `$_POST` and uploaded files into `$_FILES`.

> **Note:** If you're developing any non-Aphiria applications on your web server, use <a href="http://php.net/manual/en/configuration.file.per-user.php" target="_blank">_.user.ini_</a> to limit this setting to only your Aphiria application.  Alternatively, you can add `php_value enable_post_data_reading 0` to an _.htaccess_ file or to your _httpd.conf_.

<h2 id="installing">Installing</h2>

Aphiria comes with a <a href="https://github.com/aphiria/app" target="_blank">skeleton app</a> to help you quickly get up and running.  It can be installed using Composer:

```bash
composer create-project aphiria/app --prefer-dist --stability dev
```

Be sure to [configure your server](#server-config) to finish the installation.

> **Note:** You can <a href="https://getcomposer.org/download/" target="_blank">download Composer from here</a>.

<h3 id="libraries">Libraries</h3>

Aphiria is broken into various libraries, each of which can be installed individually:

* `aphiria/api`
* `aphiria/application`
* `aphiria/authentication`
* `aphiria/authorization`
* `aphiria/collections`
* `aphiria/console`
* `aphiria/content-negotiation`
* `aphiria/dependency-injection`
* `aphiria/exceptions`
* `aphiria/framework`
* `aphiria/io`
* `aphiria/middleware`
* `aphiria/net`
* `aphiria/psr-adapters`
* `aphiria/reflection`
* `aphiria/router`
* `aphiria/security`
* `aphiria/sessions`
* `aphiria/validation`

<h2 id="server-config">Server Config</h2>

* Aphiria's _tmp_ directory needs to be writable from PHP
* The document root needs to be set to Aphiria's _public_ directory (usually _/var/www/html/public_ or */var/www/html/YOUR_SITE_NAME/public*)

> **Note:** You must set `YOUR_SITE_DOMAIN` and `YOUR_SITE_DIRECTORY` with the appropriate values in the configs below.

<h3 id="php-built-in-web-server-config">PHP Built-in Web Server Config</h3>

To run Aphiria locally, run the following in a terminal:

```php
php aphiria app:serve
```
    
This will run PHP's built-in web server. The site will be accessible at http://localhost:8080.

<h3 id="apache-config">Apache Config</h3>

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

<h3 id="nginx-config">Nginx Config</h3>

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
        fastcgi_pass            unix:/run/php/php8.2-fpm.sock;
    }
}
```

<h3 id="caddy-config">Caddy Config</h3>

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
