---
title: "The Day WordPress Almost Broke"
date: "2025-09-17"
description: "From 10-Second Page Loads to 1.6M Requests per Hour: Our WordPress Scaling Story."
draft: false
tags: ["Nginx", "Cache", "Fastcgi", "Wordpress"]
---

## Introduction
Every WordPress site eventually runs into scaling challenges. Our story started with what seemed like plenty of resources, but as traffic grew, we learned the hard way that hardware alone doesn’t guarantee performance.

Here’s how we went from painfully slow load times to serving millions of requests per hour efficiently.

## The Bottleneck: When Hardware Isn’t Enough
Our first setup was a single large server: 16 virtual CPUs, 32 GB of RAM, and 200 GB of storage. It ran WHM/cPanel with Apache handling the web layer and PHP-FPM running WordPress.

For a while it worked fine. But one day, traffic spiked dramatically and the homepage suddenly took around 10 seconds to load.

When we checked the metrics:

- All 16 CPU cores were at 100% usage constantly
- PHP-FPM workers were maxed out
- MySQL was struggling under load

We tried the usual fixes: tuning PHP-FPM process management, tweaking MySQL buffer sizes, optimizing queries - but nothing moved the needle. The site remained slow, even though the server had powerful specs.

It became clear: the problem wasn’t lack of CPU or RAM, but how the stack was serving requests.

## Weeks of Debugging That Led Nowhere
For almost **two to three weeks**, we were stuck in a cycle of testing, tweaking, and hoping the next change would finally fix the slowdown. Since the server was running on WHM and cPanel, we started our investigation there. The built-in dashboards showed us what we already knew: CPU usage was constantly at 100% and memory consumption was stable, but nothing looked obviously misconfigured. We combed through Apache and PHP settings in WHM, checked resource limits, and made sure no hidden bottleneck or redirect was eating up performance.

Next, we turned our attention to PHP-FPM. We experimented with every process management mode available—dynamic, ondemand, even static—trying to find a sweet spot. We increased the number of children, adjusted memory allocation, and fine-tuned the spare servers. Each tweak gave us hope, but in the end, the CPUs stayed maxed out no matter what combination we tried.

MySQL was another prime suspect. We tuned InnoDB parameters, increased buffer sizes, reviewed slow query logs, and even removed some heavy plugins that generated inefficient queries. While those changes gave us slight improvements when traffic was low, the database simply couldn’t keep up once requests surged.

We even suspected .htaccess for a while. Apache relies heavily on it, and we thought maybe too many rewrite rules or unnecessary redirects were dragging the site down. After cleaning it up and testing multiple times, everything looked clean—redirects were fast and working properly—but the site still crawled when visitors poured in.

By this point we had also tried several caching plugins inside WordPress, hoping they would give us a quick win. They helped a little, but since Apache and PHP still had to process every request before serving the cache, it wasn’t enough to deal with high concurrency.

After weeks of this, the result was always the same: the homepage took nearly 10 seconds to load and all 16 CPU cores were permanently stuck at 100%. We weren’t dealing with a misconfiguration—it was the architecture itself holding us back. Apache with .htaccess and WordPress without a proper external caching layer simply couldn’t scale.

That realization was the turning point. Instead of fighting with the old stack, we decided to take a fresh approach on a completely new server.

## The Switch: Clean Slate on a Smaller Server
Instead of endlessly patching the old environment, we built a new VM with smaller specs (8 CPUs, 16 GB RAM). The idea was to test a more modern, efficient stack.

This time, we:

- Installed WordPress with PHP 8.3 and a fresh MySQL instance
- Migrated the site’s files and database with rsync
- Replaced Apache with Nginx

The most important change was enabling FastCGI caching in Nginx. Instead of regenerating every page through WordPress on every request, Nginx could serve cached versions instantly, only hitting PHP/MySQL when necessary.

The result was immediate: the site went from sluggish to lightning fast, even under traffic spikes.

## The Optimizations
To make the new server production-ready, I added several key optimizations:

### PHP-FPM Configuration

I configured PHP-FPM in static mode with 48 children so it’s always ready to handle spikes in traffic:
```ini
[www]
user = www-data
group = www-data
listen = /run/php/php8.3-fpm.sock
listen.owner = www-data
listen.group = www-data

pm = static
pm.max_children = 48
pm.start_servers = 16
pm.min_spare_servers = 12
pm.max_spare_servers = 24
pm.max_requests = 1000

php_admin_value[memory_limit] = 256M
```
This ensures the pool doesn’t waste time spawning processes during load surges.

### Database Optimizations

I also reconfigured MySQL for high-load performance, focusing on InnoDB:
```ini
# Disable binary logging
skip-log-bin

## InnoDB tuning
innodb_flush_neighbors = 0
innodb_log_file_size = 1G
innodb_log_files_in_group = 2
innodb_flush_log_at_trx_commit = 2
innodb_redo_log_capacity = 4G
innodb_max_dirty_pages_pct = 75
innodb_max_dirty_pages_pct_lwm = 50
innodb_flush_method = O_DIRECT
innodb_buffer_pool_size = 8G
innodb_log_writer_threads = 2
```
This cut down on disk I/O and gave us much smoother performance under heavy write and read activity.

### Multiple Caching Layers
One of the biggest breakthroughs was building a layered caching strategy. Instead of relying only on WordPress plugins for caching, we moved caching down into the web server itself.

At the core, we enabled FastCGI cache in Nginx. This meant that once a page was generated by WordPress, Nginx could serve the cached response directly to the next visitor - bypassing PHP and MySQL entirely.

Here’s a simplified version of our Nginx configuration:
```nginx
fastcgi_cache_path /var/cache/nginx levels=1:2 keys_zone=WORDPRESS:2048m inactive=60m;
fastcgi_cache_key "$scheme$request_method$host$request_uri";

server {
    listen 80;
    server_name <DOAMIN>;
    server_tokens off;

    location / {
    	return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name <DOAMIN>;
    server_tokens off;

    root /var/www/wordpress;
    index index.php index.html index.htm;

    ssl_certificate /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    client_max_body_size 2048M;  # Support 2G upload

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ ()\.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        # Don't cache URIs
        if ($request_uri = "/") { set $skip_cache 1; }
        if ($request_uri ~* "^/wp-admin/") { set $skip_cache 1; }
        if ($request_uri = "/wp-login.php") { set $skip_cache 1; }
        if ($request_uri = "/wp-json/") { set $skip_cache 1; }

        # Don't cache wordpress cookies
        if ($http_cookie ~* "comment_author") { set $skip_cache 1; }
        if ($http_cookie ~* "wordpress_[a-f0-9]+") { set $skip_cache 1; }
        if ($http_cookie ~* "wp-postpass") { set $skip_cache 1; }
        if ($http_cookie ~* "wordpress_no_cache") { set $skip_cache 1; }
        if ($http_cookie ~* "wordpress_logged_in") { set $skip_cache 1; }
        if ($http_cookie ~* "postpass") { set $skip_cache 1; }
        if ($http_cookie ~* "wordpress_n$") { set $skip_cache 1; }

        # Cache
        fastcgi_cache WORDPRESS;
        fastcgi_cache_valid 200 60m;  # Cache 200 OK responses for 60 minutes
        fastcgi_cache_valid 404 1m;   # Cache 404s for 1 minute
        fastcgi_cache_use_stale error timeout updating;
        fastcgi_cache_background_update on;
        fastcgi_cache_revalidate on;
        fastcgi_ignore_headers Cache-Control Expires Set-Cookie;
        fastcgi_cache_bypass $skip_cache;
        fastcgi_no_cache $skip_cache;

        add_header X-Cache $upstream_cache_status;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|mp4)$ {
        expires max;
        log_not_found off;
    }
}
```
This setup ensures that:

- Visitors see cached content instantly (most requests never touch PHP or MySQL).
- Logged-in users and admins are excluded from cache, so they always see fresh content.
- Static assets like JS, CSS, images, and MP4s are served with long cache lifetimes.
- A custom header (X-Cache) lets us see whether a request was served from cache or generated fresh.

On top of this:

- Redis Object Cache handled query-level caching inside WordPress.
- CDN offloaded static media to reduce bandwidth and latency.
- With these three layers working together - FastCGI cache, Redis cache, and CDN - the majority of traffic never even reached the database.

## Results and Lessons Learned
After moving to the new server and applying all the optimizations, the difference was huge. Pages that used to take ~10 seconds now load almost instantly, and the site can handle around 1.6 million requests per hour. CPU usage that once maxed out 16 cores now sits around 5 cores, and memory usage is only 4 GB. We are running faster with fewer resources.

The process also taught us some important lessons:

- **More hardware isn’t always the solution:** Our old 16-core server struggled, while the new optimized 8-core server handles more traffic easily.
- **Caching is key:** FastCGI cache, Redis object cache, and a CDN together reduced the load on PHP and MySQL dramatically.
- **Nginx scales better than Apache:** Removing .htaccess and using FastCGI caching made performance predictable under heavy load.
- **Databases need tuning:** Optimizing MySQL and caching repeated queries with Redis made the database stable under high traffic.
- **Static PHP-FPM pools help:** Pre-spawned workers ensured the server was always ready for busy periods without delays.

What started as a frustrating slow site turned into a lean, fast, and scalable system. Now, instead of worrying about traffic spikes, we’re ready for growth.
