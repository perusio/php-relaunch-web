# Shell script to restart php-fpm when a 502 or a timeout occurs

## Introduction 

This is a shell script that should be run as root that restarts
[php-fpm](http://www.php.net/manual/en/install.fpm.php) when a **502
Bad Gateway** error or there's a timeout when requesting a PHP page
using cURL.

This is not to be used as **permanent** fix, but rather a placeholder
that is to be used while you're tuning your php-fpm settings. If this
script invoked frequently then there's something **very wrong** with
your site.

The script works by requesting every **n** seconds a very simple
[PHP page](https://github.com/perusio/php-heartbeat) that just prints
**two** variables: `$_SERVER['HTTP_HOST']` and
`$_SERVER['SERVER_ADDR']`.

## Installation

 1. Clone the [repo](https://github.com/perusio/php-relaunch-web.git).

 2. Clone the [PHP page](https://github.com/perusio/php-heartbeat.git).

 3. Configure your server for supporting this script. Here's an example
    [Nginx](http://wiki.nginx.org) config:
               
        server {
           listen [::]:8889;
           server_name heartbeat-no-ip;

           access_log /var/log/nginx/heartbeat.access.log;
           error_log /var/log/nginx/heartbeat.error.log;

           ## If the hearbeat is not allowed close the connection.
           if ($heartbeat_not_allowed) {
               return 444;
           }

           root /var/www/sites/php-heartbeat;
           index index.php;

           ## The catch all location.
           location / {
               empty_gif;
           }

           ## The PHP heartbeat.
           location = /php-heartbeat.php {
               access_log off;
               fastcgi_pass phpcgi;
           }
        } # server
     
 5. Is **your** responsability to restrict this host to be accessed
    on the loopback or LAN. You can use something like this:
    <pre>
    geo $heartbeat_not_allowed {
        default 1;
        127.0.0.1 0;
        192.168.65.0/24 0;
    } 
    </pre>
 6. Install the script in the `crontab` of `root`:
     
    `* * * * * /path/to/php-relaunch http://heartbeat.no-ip:8889/php-heartbeat.html 5 'some@address.com' &>/dev/null`
    
    This sets a timeout of **5** seconds when requesting the above
    test page. If a restart happens then email an alert to
    `some@address.com`. Besides sending an emai, which is optional,
    just pass the empty string "" as address, a log is always recorded
    in `/tmp/php_relaunch.log`.
    
 6. If you're running `php-cgi` as a fallback to `php-fpm` then pass an
    additional argument to the script. The above `crontab` entry would
    be:
       
    `* * * * * /path/to/php-relaunch http://heartbeat.no-ip:8889/php-heartbeat.html 5 'some@address.com' 1 &>/dev/null`
 
 7. Done.
