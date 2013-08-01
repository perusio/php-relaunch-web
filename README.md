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

## Requirements 

The script included requires the following:

 + `ts` utility included on the debian [moreutils]() package.
 
 + `flock` included in the debian [util-linux]() package.

 + `curl` command line HTTP client.
 
 + `service` script included on the debian [sysvinit-utils]() package.
 
 + `pgrep` included on the debian [procps]() package.

If any of the above is not present the script will **fail**.

## Installation

 1. Clone the [repo](https://github.com/perusio/php-relaunch-web.git).

 2. Clone the [PHP page](https://github.com/perusio/php-heartbeat.git).

 3. Configure your server for supporting this script. In the `nginx`
    directory a suggested configuration is given. The vhost
    configuration is: 

    [Nginx](http://wiki.nginx.org) config:
               
        server {
           listen 127.0.0.1:8889; # IPv4 loopback bound
           listen [::1]:8889 ipv6only=on; # IPv6 loopback bound
           server_name heartbeat.no-ip;

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
     
 4. Is **your** responsability to restrict access to this host.  In
    the `nginx` directory there's a `heartbeat_allowed.conf` file that
    should be included at the `http` level of your nginx
    configuration. It restricts the hearbeat access to the loopback 
    or/and a (V)LAN. Here's an example with the loopback and a IPv4
    and IPv6 VLAN: 
    <pre>
    geo $heartbeat_not_allowed {
        default 1;
        127.0.0.1 0;
        ::1 0;
        fd0c:b7ed:0666:33a3::/64 0;
        192.168.65.0/24 0;
    } 
    </pre>
    
 5. Install the script in the `crontab` of `root`:
     
    `* * * * * /path/to/php-relaunch http://heartbeat.no-ip:8889/php-heartbeat.php 5 'some@address.com' &>/dev/null`
    
    This sets a timeout of **5** seconds when requesting the above
    test page. If a restart happens then email an alert to
    `some@address.com`. Besides sending an emai, which is optional,
    just pass the empty string "" as address, a log is always recorded
    in `/tmp/php_relaunch.log`.
    
 6. If you're running `php-cgi` as a fallback to `php-fpm` then pass an
    additional argument to the script. The above `crontab` entry would
    be:
       
    `* * * * * /path/to/php-relaunch http://heartbeat.no-ip:8889/php-heartbeat.php 5 'some@address.com' 1 &>/dev/null`
 
 7. Done.

Note that you can have **multiple** email addresses. Just add them between
quotes, separated by a comma, for **two** addresses:

    "some@address1.com,another@address2.com"
    
Here's the full crontab line:

    `* * * * * /path/to/php-relaunch
    http://heartbeat.no-ip:8889/php-heartbeat.php 5 'some@address.com,someoneelse@otheraddress.com' &>/dev/null`
    
# TODO

 + Add a **simplified** version to work with [mcron](http://www.gnu.org/software/mcron/).
   
 + Add a FreeBSD oriented setup. Pull requests very much welcomed.  

