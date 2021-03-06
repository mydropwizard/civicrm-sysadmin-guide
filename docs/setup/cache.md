A *cache* stores volatile, ephemeral data.  CiviCRM has multiple caches which may be stored in different ways, depending on the particular data and the environment/configuration.

CiviCRM's default configuration is designed to minimize system-requirements and maximize compatibility -- you only need PHP and MySQL.  This simplifies installation on self-hosted infrastructure.

However, if you'd like to *improve the performance of the system*, you may install and configure a *specialized cache service* such as *Redis* or *Memcache*.  These cache services improve performance by keeping more data in short-term RAM and by omitting traditional data-management features.  They're ideally suited to volatile data which may be heavily used for a short time.

!!! note "How much does performance improve with a specialized cache service?"

    The exact answer will depend on the use-case and environment. However, we can consider an example.

    I installed CiviCRM `5.6.alpha1` on a barebones, local Drupal 7 site and benchmarked the time it takes to render the "Event Info" page (`/civicrm/event/info?id=1`) with the [Apache "ab"](https://en.wikipedia.org/wiki/ApacheBench) command.  This sends a large number of HTTP requests and calculates the average response time.

    ```bash
    $ ab -n 100 -c 3 'http://localhost/civicrm/event/info?id=1'
    ```

    With the default, pure-MySQL configuration, the mean response time was ~460ms.  With Redis, this decreased by ~60ms (~13%) to ~400ms.
    
    Note, though, the local copy of MySQL on this laptop is probably faster than a typical MySQL because it stores *everything* in ram-disk (rather than HDD or SSD). Compared to a typical, self-hosted MySQL, the Redis/Memcache advantage is probably wider.

!!! note "What data (exactly) is stored where (exactly)?"

    This is a good but complex question. Generally speaking, the "caches" may store:

    * Metadata about local configuration (e.g. extensions, settings, option-values)
    * Session-state describing active users and their forms
    * Computed values for "Smart group" memberships and access controls
    * Query results generated by the "Advanced Search" and displayed over multiple pages
    * Response-data from external web-services

    Some of these are flexible caches with swappable backends, and some are specialized and hard-coded. The exact details could fill several more pages.  For our purposes in the *System Administration Guide*, the key question is this: *What decisions and options are available to the sysadmin to improve performance?*

## Choose a cache service {:#choose}

CiviCRM integrates with the following specialized cache services:

* [Memcached](https://en.wikipedia.org/wiki/Memcached) is well-known for defining the genre of minimalist memory-backed cache services, and it remains true to its original minimalism.
* [Redis](https://en.wikipedia.org/wiki/Redis) is also a lightweight memory-backed cache service, but it adds a few features for system administration (eg basic access-control; optional logging/persistence) and development (eg richer cache APIs).
* APC(u) is a local service bundled with some PHP servers. There is generally less literature on managing it, but it may be suitable for some single-server deployments. It is *not* suitable if you have multiple web-servers.

These services are used by many PHP applications, so your hosting provider may already support one of them.

If you're free to choose and have no particular preference, then **use Redis**. It provides a good balance of performance and functionality.

## Install the cache service and driver {:#install}

For Redis or Memcached, you will need to install the server and a PHP driver, e.g.

* [Redis server](https://redis.io/) and [php-redis driver](https://github.com/phpredis/phpredis)
* [Memcached server](https://memcached.org/) and [php-memcached driver](http://php.net/manual/en/book.memcached.php) (or the alternate [php-memcache driver](http://php.net/manual/en/book.memcache.php))

The servers and drivers are available in many formats and channels (`apt-get`, `yum`, etc).  There are a number of tutorials on the Internet for specific environments, e.g.

* *Server+Driver*: [How to install Redis and Redis php client on Ubuntu](https://anton.logvinenko.site/en/blog/how-to-install-redis-and-redis-php-client.html)
* *Server*: [Redis QuickStart](https://redis.io/topics/quickstart)
* *Server*: [How to Install Redis on macOS El Capitan, Sierra, et al](https://medium.com/@djamaldg/install-use-redis-on-macos-sierra-432ab426640e)
* *Driver*: [Redis extension for MAMP 3.x & 4.x](https://github.com/panxianhai/php-redis-mamp)

Once you've installed, you should have a few pieces of information:

* *Redis*: Hostname (e.g. `localhost`), port (e.g `6379`), and optionally a password
* *Memcache*: Hostname (e.g. `localhost`), port (e.g `11211`)

## Configure the CiviCRM cache {:#config}

The cache service is an essential part of CiviCRM that must be available at the very beginning of each page-request.  To configure CiviCRM to use a cache, edit the `civicrm.settings.php` and update the `define()` statements for `CIVICRM_DB_CACHE_*` options.  Below are some examples followed by a more detailed reference.

### Example: Memcache {:#config-memcache}

```php
define('CIVICRM_DB_CACHE_CLASS', 'Memcached'); // Or the alternate 'Memcache'
define('CIVICRM_DB_CACHE_HOST', '127.0.0.1');
define('CIVICRM_DB_CACHE_PORT', 11211);
define('CIVICRM_DB_CACHE_PREFIX', 'example.com');
```

### Example: Redis {:#config-redis}

```php
define('CIVICRM_DB_CACHE_CLASS', 'Redis');
define('CIVICRM_DB_CACHE_HOST', '127.0.0.1');
define('CIVICRM_DB_CACHE_PORT', 6379);
define('CIVICRM_DB_CACHE_PASSWORD', 'topsecret');
define('CIVICRM_DB_CACHE_PREFIX', 'example.com');
```

### Example: Redis and civibuild {:#config-redis-civibuild}

If you are a developer using `civibuild` to manage several local dev sites, you can create one file `/etc/civicrm.settings.d/pre.d/100-cache.php` to activate Redis on all sites.  

```php
if (!defined('CIVICRM_TEST') || !(getenv('CIVICRM_UF') === 'UnitTests')) {
  define('CIVICRM_DB_CACHE_CLASS', 'Redis');
  define('CIVICRM_DB_CACHE_HOST', '127.0.0.1');
  define('CIVICRM_DB_CACHE_PORT', 6379);
  define('CIVICRM_DB_CACHE_PASSWORD', 'topsecret');
  define('CIVICRM_DB_CACHE_PREFIX', $civibuild['SITE_NAME']);
}
```

!!! tip "Cache service and automated testing"

    Developers often run automated tests.  The cache-service should be enabled for  *regular web-browsing* and [end-to-end](https://docs.civicrm.org/dev/en/latest/testing/#e2e) testing, but it should not be enabled for [headless](https://docs.civicrm.org/dev/en/latest/testing/#headless) testing.  This is why we have the extra guard for `CIVICRM_TEST` and `CIVICRM_UF`.

### Reference {:#config-ref}

* `CIVICRM_DB_CACHE_CLASS`: Specifies the type of driver. The following values are allowed:
    * `APCcache` - Use the PHP APC or APCu extension
    * `ArrayCache` (*default*) - Use an temporary, in-memory cache (unique for each page-request).
    * `Memcache` - Use the PHP `memcache` extension
    * `Memcached` - Use the PHP `memcached` extension
    * `NoCache` - Do not cache anything
    * `Redis` - Use the PHP `redis` extension
* `CIVICRM_DB_CACHE_HOST`: The hostname or IP address of the remote server. (Applies to `Memcache`, `Memcached`, `Redis`.)
* `CIVICRM_DB_CACHE_PORT`: The port number of the remote server. (Applies to `Memcache`, `Memcached`, `Redis`.)
* `CIVICRM_DB_CACHE_PASSWORD`: A secret that grants access to the cache-server. (Only applies to `Redis`.)
* `CIVICRM_DB_CACHE_PREFIX`: A unique prefix that differentiates your CiviCRM cache. (Applies to `APCcache`, `Memcache`, `Memcached`, `Redis`.)
    * The prefix is strongly recommended but not strictly required. Setting a unique prefix serves a few purposes:
        * If the cache server is shared with other applications, the prefix prevents collisions between applications.
        * If the cache server is used in a multisite/multienant environment, the prefix prevents collisions between sites.
        * When debugging or inspecting a cache, the prefix helps you identify the origin of each cache record.
