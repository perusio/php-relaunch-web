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
 
 + [`mcron`](http://www.gnu.org/software/mcron/) if you want a
   **high** precision cron (second granularity).

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
 
## Using mcron

[`mcron`](http://www.gnu.org/software/mcron/) is a high-precision job
scheduler. It's to be preferred to `cron` &mdash; aka Vixie's cron
&mdash; whenever there's the need for executing jobs in the
**seconds** range.

`mcron` is written in [Guile](), a dialect of Scheme, and the
preferred way of configuring it. There's a Vixie's cron compatible
configuration, but using it you cannot specify complex execution
conditions or second level timings. Hence here we use a Guile (Scheme)
job specification. Here are the things to bear in mind when using
`mcron`:

 + `mcron` is run for **each** user, so we need to launch it in 
   **daemon** mode.

 + It relies on a `~/.cron` directory where there's a `job.guile` file
   specifying all jobs.
   
 + Since in order to restart `php-fpm` we need to be root, `mcron`
   must be run as **root**.
   
Here's the howto for running the `php-relaunch` script using `mcron`: 

 1. Repeats steps 1 to 4 of the Installation procedure above:
 
 2. Create a `/root/.cron` directory.
 
 3. Create a file `job.guile` in the `/root/.cron` directory and add
    the Guile code: 
    
        (job '(next-second (range 0 60 10)) "/path/to/php-relaunch http://heartbeat.no-ip:8889/php-heartbeat.php 5 'some@address.com' &>/dev/null")
 
    This means that the job will execute **every 10 seconds**. Which
    makes sense given that we passed as argument to `php-relaunch` a
    value of **5** seconds as the threshold for considering when PHP
    is unresponsive.   
 
 4. Launch `mcron` in daemon mode as root:
    
        mcron -d
 
 5. Done. 
 
In the `mcron` directory there's an example `job.guile` use it for
creating your own. Adapt the path to `php-relaunch`, to the URI of the
PHP heartbeat page, threshold for considering PHP unresponsive and the
email address(es) to be notified when a relaunch occurs.p
 
## mcron with Vixie's cron as a fallback
 
We can setup **both** types of cron utilities: try to use `mcron`
first and fallback on Vixie's cron in case it's not running.
 
For that we configure **both** `mcron` and `cron`. The only thing we
need to change is the crontab line to:
 
    `* * * * * pgrep mcron || (/path/to/php-relaunch http://heartbeat.no-ip:8889/php-heartbeat.php 5 'some@address.com' &>/dev/null`)
 
or for multiple email addresses being notified: 
 
    `* * * * * pgrep mcron || (/path/to/php-relaunch http://heartbeat.no-ip:8889/php-heartbeat.php 5 'some@address.com,someoneelse@otheraddress.com' &>/dev/null`)     
 
 Now whenever mcron is not running we launch `php-relaunch` every minute. 
 
## TODO

 + Add a FreeBSD oriented setup. Pull requests very much welcomed.  

