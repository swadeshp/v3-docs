# 📌 Requirements

::: tip Docker
Refer to the official [Docker images for Chevereto](https://github.com/chevereto/docker) for the recommended system setup to run Chevereto, including all libraries required.
:::

* PHP
* Database (MySQL / MariaDB)
* Cron setup
* HTTP web Server (Nginx / Apache / *any*)

## PHP

| Version | PHP |
| ------- | --- |
| 3.19    | 7.4 |

### Extensions

The following PHP extensions are required for Chevereto.

* imagick with `PNG GIF JPG BMP WEBP`
* curl
* hash
* json
* mbstring
* pdo
* pdo-mysql
* zip
* session
* xml
* fileinfo

### Filesystem

User running `php` must be in the owner group of your installation directory. This is required to allow Chevereto to modify the filesystem for uploading, one-click update and many other features.

> PHP usually runs under `www-data`

Chevereto user will require **read/write** access in the following paths:

* `/tmp`
* `app/content/`
* `app/content/languages/`
* `app/content/languages/cache/`
* `app/content/system/`
* `content/`
* `images/`

## Database

| Version | MySQL | MariaDB |
| ------- | ----- | ------- |
| 3.19    | 8     | 10      |

MySQL/MariaDB user must have `ALL PRIVILEGES` over the target database.

## Cron

A cron is required to process the application background jobs. The cron for your installation may look like this:

```sh
* * * * * IS_CRON=1 /usr/bin/php /var/www/html/chevereto.loc/public_html/cron.php
```

Where [* * * * *](https://crontab.guru/#*_*_*_*_*) is the cron schedule to run every minute.

### Run cron

You can go to [explainshell](https://explainshell.com/explain?cmd=IS_CRON%3D1+%2Fusr%2Fbin%2Fphp+%2Fvar%2Fwww%2Fhtml%2Fchevereto.loc%2Fpublic_html%2Fcron.php+%3E%2Fdev%2Fnull+2%3E%261) to inspect the command, you can freely alter it to match your needs. Run the command as `www-data` user by adding `sudo -u www-data` to the command:

```sh
sudo -u www-data IS_CRON=1 /usr/bin/php /var/www/html/chevereto.loc/public_html/cron.php
```

### Run cron using Docker

When using Docker is recommended that the cron setup executes outside the container.

You can achieve that by running this from the host:

```sh
docker exec -it \
    --user www-data \
    -e IS_CRON=1 \
    my-container /usr/local/bin/php /var/www/html/cron.php
```

## Web server

### URL rewriting

The web server must rewrite HTTP requests like `GET /image/some-name.<id>` to `/index.php`. Instructions for [Nginx](https://nginx.org/) and [Apache HTTP Server](https://httpd.apache.org/) below.

#### Nginx URL rewriting

`example.com.conf`

```nginx
# Context limits
client_max_body_size 25M;

# Disable access to sensitive files
location ~* (app|content|lib)/.*\.(po|php|lock|sql)$ {
    deny all;
}

# Image not found replacement
location ~ \.(jpe?g|png|gif|webp)$ {
    log_not_found off;
    error_page 404 /content/images/system/default/404.gif;
}

# CORS header (avoids font rendering issues)
location ~ \.(ttf|ttc|otf|eot|woff|woff2|font.css|css|js)$ {
    add_header Access-Control-Allow-Origin "*";
}

# Pretty URLs
location / {
    index index.php;
    try_files $uri $uri/ /index.php$is_args$query_string;
}
```

#### Apache HTTP Server URL rewriting

Make sure that [`mod_rewrite`](https://httpd.apache.org/docs/current/mod/mod_rewrite.html) is enabled and that your virtual host settings allows to perform URL rewriting:

```apacheconf
    <Directory /var/www/html>
        Options -Indexes +FollowSymLinks +MultiViews
        AllowOverride All
        Require all granted
    </Directory>
```

Apache configuration `.htaccess` files are already included in the software.

`/.htaccess`

```apacheconf
# Disable server signature
ServerSignature Off

# Enable CORS across all your subdomains (replace dev\.local with your domain\.com)
# SetEnvIf Origin ^(https?://.+\.dev\.local(?::\d{1,5})?)$   CORS_ALLOW_ORIGIN=$1
# Header append Access-Control-Allow-Origin  %{CORS_ALLOW_ORIGIN}e   env=CORS_ALLOW_ORIGIN
# Header merge  Vary "Origin"

# Disable directory listing (-indexes), Multiviews (-MultiViews)
Options -Indexes
Options -MultiViews

<IfModule mod_rewrite.c>

    RewriteEngine On

    # If you have problems with the rewrite rules remove the "#" from the following RewriteBase line
    # You will also have to change the path to reflect the path to your Chevereto installation
    # If you are using alias is most likely that you will need this.
    #RewriteBase /

    # 404 images
    # If you want to have your own fancy "image not found" image remove the "#" from RewriteCond and RewriteRule lines
    # Make sure to apply the correct paths to reflect your current installation
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule images/.+\.(gif|jpe?g|png|bmp|webp) - [NC,L,R=404]
    #RewriteRule images/.+\.(gif|jpe?g|a?png|bmp|webp) content/images/system/default/404.gif [NC,L]

    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_URI} !\.(css|js|html|htm|rtf|rtx|svg|svgz|txt|xsd|xsl|xml|asf|asx|wax|wmv|wmx|avi|bmp|class|divx|doc|docx|exe|gif|gz|gzip|ico|jpe?g|jpe|mdb|mid|midi|mov|qt|mp3|m4a|mp4|m4v|mpeg|mpg|mpe|mpp|odb|odc|odf|odg|odp|ods|odt|ogg|pdf|png|pot|pps|ppt|pptx|ra|ram|swf|tar|tif|tiff|wav|webp|wma|wri|xla|xls|xlsx|xlt|xlw|zip)$ [NC]
    RewriteRule . index.php [L]

</IfModule>
```

### Real connecting IP

For setups under any kind of proxy (including CloudFlare) is required that the web server sets the appropriate value for the client connecting IP.

::: warning
If this is not configured the software won't be able to detect the visitors IPs, failing to deliver IP based restrictions and flood control.
:::

* Nginx: `ngx_http_realip_module`
* Apache: `mod_remoteip`
