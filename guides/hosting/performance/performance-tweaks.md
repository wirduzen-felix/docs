# Performance Tweaks

Shopware is a platform for many different projects. It needs to handle a broad range of load characteristics and environments. That means that the default configuration is optimized for the best out-of-the-box experience. But there are many opportunities to increase the performance by fitting the configuration to your needs.

## HTTP Cache

To ensure a high RPS (requests per second), Shopware offers an integrated HTTP cache with a possible reverse proxy configuration. Any system that handles high user numbers should always use HTTP caching to reduce server resources.

To enable this set `SHOPWARE_HTTP_CACHE_ENABLED=1` in the `.env`

### Reverse Proxy Cache

When you have a lot of app servers, you should consider using a reverse proxy cache like Varnish. Shopware offers default configuration for Varnish out of the box.

[Read more](../infrastructure/reverse-http-cache.md)

### logged-in / cart-filled

By default, Shopware can no longer deliver complete pages from a cache for a logged-in customer or if products are in the shopping cart. As soon as this happens, the user sessions differ, and the context rules could be different depending on the user. The result is different content for each customer; a good example is the [Dynamic Access](https://store.shopware.com/swag140583558965t/dynamic-access.html) plugin.

However, if the project does not require such functionality, pages can also be cached by the HTTP cache/reverse proxy. To disable Cache invalidation in these cases:

```yaml
shopware:
    cache:
        invalidation:
            http_cache: []
```

### Delayed invalidation

A delay for the cache invalidation can be activated for systems with a high update frequency for the inventory (products, categories). Once the instruction to delete the cache entries for a specific product or category occurs, they are not deleted instantly but processed by a background task afterwards. Thus, if two processes invalidate the cache in quick succession, the timer for the invalidation of this cache entry will only reset.

```yaml
shopware:
    cache:
        invalidation:
            delay: 0
            count: 150
```

## MySQL instead of MariaDB

In some places in the code, we use JSON fields. As soon as it comes to filtering, sorting, or aggregating JSON fields, MySQL is ahead of the MariaDB fork. Therefore, we strongly recommend the use of MySQL.

## MySQL configuration

Shopware sets some MySQL configuration variables on each request to ensure it works in any environment. You can disable this behavior if you have correctly configured your MySQL server.

- Make sure that `group_concat_max_len` is by default higher or equal to `320000`
- Make sure that `sql_mode` doesn't contain `ONLY_FULL_GROUP_BY`

and then you can set `SQL_SET_DEFAULT_SESSION_VARIABLES=0` to your `.env` file

## SQL is faster than DAL

We designed the DAL (Data Abstraction Layer) to provide developers a flexible and extensible data management. However, features in such a system come at the cost of performance. Therefore, using DBAL (plain SQL) is much faster than using the DAL in many scenarios, especially when it comes to internal processes, where often only one ID of an entity is needed.

[Read more](https://github.com/shopware/platform/blob/trunk/adr/dal/2021-05-14-when-to-use-plain-sql-or-dal.md)

## Elasticsearch

Elasticsearch is a great tool to reduce the load of the MySQL server. Especially for systems with large product assortments, this is a must-have since MySQL simply does not cope well above a certain assortment size.

When using Elasticsearch, it is important to set the `SHOPWARE_ES_THROW_EXCEPTION=1` `.env` variable. This ensures that if an error occurs when querying the data via Elasticsearch, there is no fallback to the MySQL server. In large projects, failure of the Elasticsearch leads to the MySQL server being completely overloaded otherwise.

[Read more](../infrastructure/elasticsearch/elasticsearch-setup.md)

## Prevent mail data updates

{% hint style="info" %}
This feature will be available starting with Shopware 6.4.11.0.
{% endhint %}

To provide autocompletion for the different mail templates in the administration UI, Shopware has a mechanism that writes an example mail into the database when sending the mail.

With the `shopware.mail.update_mail_variables_on_send` configuration, you can disable this source of database load:

```yaml
shopware:
    mail:
        update_mail_variables_on_send: false
```

[Read more](https://github.com/shopware/platform/blob/trunk/adr/2022-03-25-prevent-mail-updates.md)

## Increment storage

The increment storage is used to store the state and display it in the Administration.
This storage increments or decrements a given key in a transaction-safe way, which causes locks upon the storage. Therefore, we recommend moving this source of server load to a separate Redis:

```yaml
shopware:
    increment:
        user_activity:
          type: 'redis'
          config:
            url: 'redis://host:port/dbindex'

        message_queue:
          type: 'redis'
          config:
            url: 'redis://host:port/dbindex'
```

If you don't need such functionality, it is highly recommended to disable this behavior by using `array` as type.

[Read more](../performance/increment.md)

## Lock storage

Shopware uses [Symfony's Lock component](https://symfony.com/doc/5.4/lock.html) to implement locking functionality.
By default, Symfony will use a local file-based lock store, which breaks in multi-machine (cluster) setups. This is avoided by using one of the [supported remote stores](https://symfony.com/doc/5.4/components/lock.html#available-stores).

```yaml
framework:
    lock: 'redis://host:port'
```

[Read more](../performance/lock-store.md)

## Number Ranges

Number Ranges provide a consistent way to generate a consecutive number sequence that is used for order numbers, invoice numbers, etc.
The generation of the number ranges is an **atomic** operation, this guarantees that the generated sequence is consecutive and no number is generated twice.

By default, the number range states are stored in the database.
In scenarios where a high throughput is required (e.g. thousands of orders per minute), the database can become a performance bottleneck, because of the requirement for atomicity.
Redis offers better support for atomic increments than the database, therefore the number ranges should be stored in Redis in such scenarios.

```yaml
shopware:
  number_range:
    increment_storage: "Redis"
    redis_url: 'redis://host:port/dbindex'
```

[Read more](../performance/number-ranges.md)

## Sending mails with the Queue

Shopware sends the mails by default synchronously. This process can take a while when the remote SMTP server is struggling. For this purpose, it is possible to handle the mails in the message queue. To enable this, add the following config to your config

```yaml
framework:
    mailer:
        message_bus: 'messenger.default_bus'
```

## PHP Config tweaks

```ini
# don't evaluate assert()
assert.active=0

# cache file_exists,is_file
opcache.enable_file_override=1

# increase opcache string buffer as shopware has many files
opcache.interned_strings_buffer=20

# disable check for BOM
zend.detect_unicode=0

# increase default realpath cache
realpath_cache_ttl=3600
```

Also PHP PCRE Jit Target should be enabled, this can checked using `php -i | grep 'PCRE JIT Target'` or looking into the phpinfo page.

For an additional 2-5% performance improvement, it is possible to provide a preload file to opcache. Preload also brings a lot of drawbacks:

- Each cache clear requires a PHP-FPM restart
- Each file change requires a PHP-FPM restart
- Extension Manager does not work

The PHP configuration would look like

```ini
opcache.preload=/var/www/html/var/cache/opcache-preload.php
opcache.preload_user=nginx
```

## Cache ID

The Shopware cache has a global cache id to clear the cache faster and work in a cluster setup. This cache id is saved in the database and will only be changed when the cache is cleared. This ensures that the new cache is used and the message queue can clean the old folder. If this functionality is not used, this cache id can also be hardcoded `SHOPWARE_CACHE_ID=foo` in the `.env` to save one SQL query on each request.

## .env.local.php

Symfony recommended that a `.env.local.php` file is used in Production instead of a `.env` file not to parse an env file on any request. If you are using a containerized environment, all those variables can also be set directly in the environment variables

## Benchmarks

In addition to the benchmarks that Shopware regularly performs with the software, we strongly recommend integrating your benchmark tools and pipelines for larger systems. A generic benchmark of a product can rarely be adapted to the individual, highly customized projects.
Tools such as [locust](https://locust.io/) or [k6](https://k6.io/) can be used for this purpose.

## Logging

Set the log level of monolog to `error` to reduce the amount of logged events. Also, limiting the `buffer_size` of monolog prevents memory overflows for long-lived jobs:

```yaml
monolog:
    handlers:
        main:
            level: error  
            buffer_size: 30
        business_event_handler_buffer:
            level: error
```

The `business_event_handler_buffer` handler logs flows. Setting it to `error` will disable the logging of flow activities that succeed.
