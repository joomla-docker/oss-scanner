# syntax=docker/dockerfile:1
# Joomla! CMS.
#
# /src    the joomla-cms git checkout (the repository named in project.yaml), with its locked Composer and npm
#         dependencies installed, its media assets built from media_source/ into media/, and a working site
#         installed into it by the CMS's own console installer. Apache serves /src through mod_php with AllowOverride
#         All, so Joomla's .htaccess files apply as they do in production. The database is MariaDB on a unix socket.
#
# Start the services with `joomla-start`. The site is then at http://127.0.0.1/, the administrator at
# http://127.0.0.1/administrator/ and the web services API at http://127.0.0.1/api/index.php/v1/. `joomla-stop` stops
# them. threat_model.md lists the accounts, the API tokens and the other helpers.
#
# The PHP and Apache base is the one Joomla's official 6.2 image is built on. Node 24 and Composer 2 come from their
# official images; the CMS needs Node 20 or later and npm 10.1 or later (package.json engines).
FROM node:24-trixie-slim AS node
FROM composer:2 AS composer
FROM php:8.4-apache-trixie

COPY --from=node /usr/local/bin/node /usr/local/bin/node
COPY --from=node /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/npm
COPY --from=composer /usr/bin/composer /usr/local/bin/composer

# MariaDB supplies the server and the client. Keep the libraries used by the PHP extensions and discard their build
# headers, as the official image does.
RUN set -eux; \
    apt-get update; \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        git mariadb-server unzip; \
    savedAptMark="$(apt-mark showmanual)"; \
    apt-get install -y --no-install-recommends \
        libfreetype-dev libicu-dev libjpeg-dev libldap-dev \
        libpng-dev libwebp-dev libzip-dev; \
    docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp; \
    docker-php-ext-configure ldap --with-libdir="lib/$(dpkg-architecture -qDEB_BUILD_MULTIARCH)"; \
    docker-php-ext-install -j"$(nproc)" exif gd intl ldap mysqli pdo_mysql zip; \
    apt-mark auto '.*' > /dev/null; \
    apt-mark manual $savedAptMark > /dev/null; \
    ldd "$(php-config --extension-dir)"/*.so \
        | awk '/=> \/(usr\/)?lib\// { so = $(NF-1); sub("^/(usr/)?", "", so); print "*" so }' \
        | sort -u | xargs -r dpkg-query --search \
        | awk 'sub(":$", "", $1) { print $1 }' | sort -u | xargs -r apt-mark manual; \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    apt-get dist-clean; \
    ln -s ../lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm; \
    ln -s ../lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx

# Production-like PHP: errors are logged to Apache's error log, not shown in responses; OPcache picks up an edited
# file at once, so a patch to /src takes effect on the next request; the upload limits allow media manager tests.
# MySQL clients reach MariaDB over its unix socket when the host is `localhost`, as the installed site does.
RUN { echo 'display_errors=Off'; \
      echo 'log_errors=On'; \
      echo 'error_reporting=E_ALL'; \
      echo 'memory_limit=256M'; \
      echo 'upload_max_filesize=32M'; \
      echo 'post_max_size=32M'; \
      echo 'opcache.validate_timestamps=1'; \
      echo 'opcache.revalidate_freq=0'; \
      echo 'sendmail_path=/usr/sbin/sendmail -t -i'; \
      echo 'mysqli.default_socket=/run/mysqld/mysqld.sock'; \
      echo 'pdo_mysql.default_socket=/run/mysqld/mysqld.sock'; \
    } > "$PHP_INI_DIR/conf.d/oss-scanner.ini"

# Apache serves /src on the loopback interface only, with .htaccess enabled, so Joomla's own libraries/.htaccess keeps
# libraries/ private as on a release. At the top level of the checkout it serves only what a Joomla release ships
# (build/build.php): the release directories, and the files index.php, configuration.php (which runs; its source is
# not served), htaccess.txt, web.config.txt, robots.txt.dist, robots.txt once a site owner creates it, LICENSE.txt and
# README.txt. Every other top-level file or directory gets 403, including one created later, such as a sitemap.xml a
# site owner adds or a file an exploit writes. This keeps the rest of a development checkout private: .git, build
# tooling and its configuration (phpunit.xml holds the database login), asset sources, tests, node_modules, and
# installation/, which a production site has removed and which on a development build would let anyone repeat the
# installation. The base image links Apache's logs to the console, which a daemonised Apache does not have; make them
# real files, so PHP and Apache errors end up in /var/log/apache2/error.log.
COPY <<'CONF' /etc/apache2/conf-available/joomla.conf
ServerName 127.0.0.1
<Directory /src>
    Options -Indexes +FollowSymLinks
    AllowOverride All
    Require all granted
    <If "%{REQUEST_FILENAME} =~ m#^/src/[^/]+$# && -f %{REQUEST_FILENAME} && %{REQUEST_FILENAME} !~ m#^/src/(index\.php|configuration\.php|htaccess\.txt|\.htaccess|web\.config\.txt|robots\.txt|robots\.txt\.dist|LICENSE\.txt|README\.txt)$#">
        Require all denied
    </If>
</Directory>
<DirectoryMatch "^/src/(?!(administrator|api|cache|cli|components|files|images|includes|language|layouts|libraries|media|modules|plugins|templates|tmp)/)[^/]+/">
    Require all denied
</DirectoryMatch>
CONF
RUN set -eux; \
    a2enmod rewrite headers expires; \
    sed -i 's@/var/www/html@/src@' /etc/apache2/sites-available/000-default.conf; \
    sed -i 's/^Listen 80$/Listen 127.0.0.1:80/' /etc/apache2/ports.conf; \
    a2enconf joomla; \
    apache2ctl configtest; \
    rm -f /var/log/apache2/*.log

# There is no mail server. This sendmail writes each email Joomla sends (password resets, registration, multi-factor
# codes, notifications) to /var/mail/joomla/, so reproductions can read the links and codes in them.
RUN mkdir -p /var/mail/joomla && chown www-data:www-data /var/mail/joomla \
    && printf '%s\n' '#!/bin/sh' 'cat > "/var/mail/joomla/$(date +%Y%m%d-%H%M%S)-$$.eml"' > /usr/sbin/sendmail \
    && chmod 0755 /usr/sbin/sendmail

# Helpers (there is no init system in the container):
#   joomla-start      starts MariaDB and Apache; safe to run again
#   joomla-stop       stops both; exits 1, naming the service, if one is still running
#   as-www-data       runs a command as www-data, the user Apache runs as and the owner of /src
#   joomla-console    cli/joomla.php as www-data
#   joomla-phpunit    PHPUnit as www-data, from /src, with the checkout's phpunit.xml
#   joomla-api-token  prints a user's web services API token, giving them one if they have none
#   joomla-permission sets a group's permission for an action on an asset (root.1, a component, an item)
COPY --chmod=0755 <<'SH' /usr/local/bin/joomla-start
#!/bin/sh
set -e
# Files root creates in these places (for example by running cli/joomla.php as root, or editing configuration.php)
# would stop the site, which runs as www-data, from writing there. Hand them back first.
chown -R www-data:www-data /src/administrator/cache /src/administrator/logs /src/cache /src/tmp /src/images /src/files \
    /var/mail/joomla
[ ! -e /src/configuration.php ] || chown www-data:www-data /src/configuration.php
mkdir -p /run/mysqld /var/log/mysql
chown mysql:mysql /run/mysqld
chown mysql:adm /var/log/mysql
# A pid file can outlive its process when the container is restarted, and the pid then belongs to another process.
stale() { pid=$(cat "$1" 2>/dev/null) || return 1; [ "$(cat "/proc/$pid/comm" 2>/dev/null)" != "$2" ]; }
if ! mariadb-admin ping --silent > /dev/null 2>&1; then
    if stale /run/mysqld/mysqld.pid mariadbd; then rm -f /run/mysqld/mysqld.pid; fi
    # The container has no syslog: the server logs to /var/log/mysql/error.log, its wrapper to start.log.
    "$(command -v mariadbd-safe || command -v mysqld_safe)" --skip-syslog --log-error=/var/log/mysql/error.log \
        >> /var/log/mysql/start.log 2>&1 &
    server=$!
    waited=0
    until mariadb-admin ping --silent > /dev/null 2>&1; do
        if ! kill -0 "$server" 2>/dev/null; then
            problem="MariaDB exited before it was ready"
        elif [ "$waited" -ge 120 ]; then
            problem="MariaDB was not ready after 60 seconds"
        else
            sleep 0.5
            waited=$((waited + 1))
            continue
        fi
        echo "$problem. The ends of its logs:" >&2
        tail -n 20 /var/log/mysql/error.log /var/log/mysql/start.log >&2 || true
        exit 1
    done
fi
# Apache is running when it answers.
answers() { curl -s -o /dev/null --max-time 2 http://127.0.0.1/; }
if ! answers; then
    if stale /var/run/apache2/apache2.pid apache2; then rm -f /var/run/apache2/apache2.pid; fi
    apachectl start
    waited=0
    until answers; do
        if [ "$waited" -ge 60 ]; then
            echo "Apache did not answer within 30 seconds. The end of /var/log/apache2/error.log:" >&2
            tail -n 20 /var/log/apache2/error.log >&2 || true
            exit 1
        fi
        sleep 0.5
        waited=$((waited + 1))
    done
fi
SH
COPY --chmod=0755 <<'SH' /usr/local/bin/joomla-stop
#!/bin/sh
# Stop Apache and MariaDB, giving each at most a minute. Exits 0 when both are stopped, including when neither was
# running. A service is stopped when no live process of it is left and it no longer answers: Apache on
# http://127.0.0.1/, MariaDB on its socket. A service still running at the end is named, with what its shutdown said and
# the end of its log, and the exit code is 1; the other service is stopped regardless. Only a pid that a pid file names
# and that is a process of the service is signalled: a pid file can outlive its process when the container is
# restarted, and the pid may then belong to an unrelated process.
status=0
alive() { [ "$(cat "/proc/$2/comm" 2>/dev/null)" = "$1" ] && ! grep -qs '^State:[[:space:]]*Z' "/proc/$2/status"; }
any_alive() { for proc in /proc/[0-9]*; do alive "$1" "${proc#/proc/}" && return 0; done; return 1; }
gone_by() { while any_alive "$1" && [ "$(date +%s)" -lt "$2" ]; do sleep 0.5; done; }

deadline=$(($(date +%s) + 60))
pid=$(cat /var/run/apache2/apache2.pid 2>/dev/null || true)
if [ -n "$pid" ] && alive apache2 "$pid"; then
    apachectl stop
    gone_by apache2 "$deadline"
fi
if any_alive apache2 || curl -s -o /dev/null --max-time 2 http://127.0.0.1/; then
    echo "joomla-stop: Apache is still running. The end of /var/log/apache2/error.log:" >&2
    tail -n 20 /var/log/apache2/error.log >&2 || true
    status=1
fi

deadline=$(($(date +%s) + 60))
pid=$(cat /run/mysqld/mysqld.pid 2>/dev/null || true)
if mariadb-admin ping > /dev/null 2>&1; then
    said=$(mariadb-admin --shutdown-timeout=60 shutdown 2>&1) || true
    said="mariadb-admin shutdown said: ${said:-nothing}"
elif [ -n "$pid" ] && alive mariadbd "$pid"; then
    kill -TERM "$pid" 2>/dev/null || true
    said="Its socket did not answer, so its process ($pid) was sent SIGTERM."
else
    said="Its socket did not answer, and /run/mysqld/mysqld.pid names no live mariadbd to signal."
fi
gone_by mariadbd "$deadline"
if any_alive mariadbd || mariadb-admin ping > /dev/null 2>&1; then
    echo "joomla-stop: MariaDB is still running. $said The end of /var/log/mysql/error.log:" >&2
    tail -n 20 /var/log/mysql/error.log >&2 || true
    status=1
fi
exit "$status"
SH
COPY --chmod=0755 <<'SH' /usr/local/bin/as-www-data
#!/bin/sh
exec runuser -u www-data -- env HOME=/tmp "$@"
SH
COPY --chmod=0755 <<'SH' /usr/local/bin/joomla-console
#!/bin/sh
cd /src && exec as-www-data php /src/cli/joomla.php "$@"
SH
COPY --chmod=0755 <<'SH' /usr/local/bin/joomla-phpunit
#!/bin/sh
cd /src && exec as-www-data /src/libraries/vendor/bin/phpunit "$@"
SH
COPY --chmod=0755 <<'PHP' /usr/local/bin/joomla-api-token
#!/usr/bin/env php
<?php
// Print a user's web services API token, as the "Joomla API Token" tab of their profile shows it. A user who has
// none gets one, as saving their profile would give them. For example:
//   curl -H "Authorization: Bearer $(joomla-api-token superuser)" http://127.0.0.1/api/index.php/v1/users
\define('_JEXEC', 1);
require '/src/configuration.php';
$config   = new JConfig();
$username = $argv[1] ?? '';
if ($username === '') {
    fwrite(STDERR, "usage: joomla-api-token <username>\n");
    exit(2);
}
// With the host `localhost`, mysqli uses the unix socket from php.ini, as the site does.
$db = new mysqli($config->host, $config->user, $config->password, $config->db);
$db->set_charset('utf8mb4');
$users    = $config->dbprefix . 'users';
$profiles = $config->dbprefix . 'user_profiles';
$query = $db->prepare("SELECT id FROM $users WHERE username = ?");
$query->bind_param('s', $username);
$query->execute();
$id = (int) $query->get_result()->fetch_column();
if (!$id) {
    fwrite(STDERR, "no user named $username\n");
    exit(1);
}
$query = $db->prepare("SELECT profile_key, profile_value FROM $profiles WHERE user_id = ? AND profile_key LIKE 'joomlatoken.%'");
$query->bind_param('i', $id);
$query->execute();
$profile = array_column($query->get_result()->fetch_all(MYSQLI_ASSOC), 'profile_value', 'profile_key');
if (empty($profile['joomlatoken.token'])) {
    $seed  = base64_encode(random_bytes(32));
    $query = $db->prepare("DELETE FROM $profiles WHERE user_id = ? AND profile_key LIKE 'joomlatoken.%'");
    $query->bind_param('i', $id);
    $query->execute();
    $query = $db->prepare("INSERT INTO $profiles (user_id, profile_key, profile_value, ordering) VALUES (?, 'joomlatoken.token', ?, 1), (?, 'joomlatoken.enabled', '1', 2)");
    $query->bind_param('isi', $id, $seed, $id);
    $query->execute();
    $profile = ['joomlatoken.token' => $seed, 'joomlatoken.enabled' => '1'];
}
if ((int) ($profile['joomlatoken.enabled'] ?? 0) !== 1) {
    fwrite(STDERR, "the API token of $username is disabled in their profile\n");
    exit(1);
}
echo base64_encode('sha256:' . $id . ':' . hash_hmac('sha256', base64_decode($profile['joomlatoken.token']), $config->secret)), "\n";
PHP
COPY --chmod=0755 <<'PHP' /usr/local/bin/joomla-permission
#!/usr/bin/env php
<?php
// Set a user group's permission for an action on an asset, as the Permissions tab of Global Configuration or of a
// component's options does. Joomla has no console command for this. For example:
//   joomla-permission root.1 core.login.api 2=1        # Registered (and the groups under it) may log in to the API
//   joomla-permission com_users core.api 6=0           # Manager may not use the users web service
//   joomla-permission com_content.article.1 core.edit 3=1
// A value of 1 allows, 0 denies, and - removes the group's setting so that it inherits again.
\define('_JEXEC', 1);
require '/src/configuration.php';
$config = new JConfig();
[$asset, $action] = [$argv[1] ?? '', $argv[2] ?? ''];
$settings = \array_slice($argv, 3);
if ($asset === '' || $action === '' || !$settings || !preg_match('/^[a-z0-9._]+$/i', $action)) {
    fwrite(STDERR, "usage: joomla-permission <asset name> <action> <group id>=<1|0|-> ...\n");
    exit(2);
}
$db = new mysqli($config->host, $config->user, $config->password, $config->db);
$db->set_charset('utf8mb4');
$assets = $config->dbprefix . 'assets';
$query = $db->prepare("SELECT rules FROM $assets WHERE name = ?");
$query->bind_param('s', $asset);
$query->execute();
$json = $query->get_result()->fetch_column();
if ($json === false || $json === null) {
    fwrite(STDERR, "no asset named $asset\n");
    exit(1);
}
$rules = json_decode($json, true) ?: [];
foreach ($settings as $setting) {
    if (!preg_match('/^(\d+)=([01-])$/', $setting, $match)) {
        fwrite(STDERR, "bad setting $setting: expected <group id>=<1|0|->\n");
        exit(2);
    }
    if ($match[2] === '-') {
        unset($rules[$action][$match[1]]);
    } else {
        $rules[$action][$match[1]] = (int) $match[2];
    }
}
$rules = array_filter($rules);
$json  = json_encode($rules);
$query = $db->prepare("UPDATE $assets SET rules = ? WHERE name = ?");
$query->bind_param('ss', $json, $asset);
$query->execute();
echo "$asset: $json\n";
PHP

# scanner contract: the checkout lives inside the image, at /src. As on a production site, the web server owns the
# Joomla directory, so the site can write its caches, logs and uploads and the files an extension installation
# creates. The scanner works as root, so git is told to trust the directory.
COPY --chown=www-data:www-data . /src
WORKDIR /src
RUN git config --system --add safe.directory /src

# The locked PHP dependencies, including the dev ones (PHPUnit, PHP CS Fixer, PHPCS, PHPStan), then the locked npm
# dependencies: Joomla's `install` script builds the media assets from media_source/ into media/. Cypress's
# browser is not downloaded: the System (browser) test suite needs a browser and a site elsewhere, and does not run here.
RUN as-www-data composer install --prefer-dist --no-interaction --no-progress \
    && as-www-data composer check-platform-reqs --no-interaction \
    && as-www-data env CYPRESS_INSTALL_BINARY=0 npm ci --no-audit --no-fund \
    && rm -rf /tmp/.cache /tmp/.composer /tmp/.config /tmp/.local /tmp/.npm

# Install a site into /src with the CMS's own console installer, on MariaDB over the unix socket, then add one user
# per core group, so access-control reproductions can act as each. Every password is "<login>-password".
#   superuser      Super Users (the installer's administrator account)
#   administrator  Administrator
#   manager        Manager
#   publisher      Publisher
#   editor         Editor
#   author         Author
#   registered     Registered
#   (anonymous)    Public: the Guest group, no login
# Databases: `joomla` for the site (user joomla / joomla, table prefix jos_) and `joomla_test` for the Integration test
# suite, whose phpunit.xml points at it.
#
# A stock site lets Super Users alone log in to the API, hold an API token and use the web services (`core.api`).
# Here the two other groups that log in to the administrator can use the API too, for what the administrator lets them
# manage, because the API checks `core.api` where the administrator checks `core.manage`: Administrator gets `core.api`
# everywhere but the installer, which the administrator denies it; Manager on banners, contacts, content, media and
# newsfeeds. Author, Editor, Publisher and Registered keep the stock setting: no API.
RUN --network=none <<'EOF'
#!/bin/bash
set -euo pipefail
joomla-start
mariadb <<'SQL'
CREATE DATABASE joomla CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE joomla_test CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'joomla'@'localhost' IDENTIFIED BY 'joomla';
GRANT ALL ON joomla.* TO 'joomla'@'localhost';
GRANT ALL ON joomla_test.* TO 'joomla'@'localhost';
SQL
as-www-data php installation/joomla.php install --no-interaction \
    --site-name='Joomla OSS Scanner' --admin-user='Super User' --admin-username=superuser \
    --admin-password=superuser-password --admin-email=superuser@example.org \
    --db-type=mysqli --db-host=localhost --db-user=joomla --db-pass=joomla --db-name=joomla --db-prefix=jos_
joomla-permission root.1 core.login.api 6=1
joomla-permission root.1 core.api 7=1
joomla-permission com_installer core.api 7=0
for component in com_banners com_contact com_content com_media com_newsfeeds; do
    joomla-permission "$component" core.api 6=1
done
mariadb joomla -e "UPDATE jos_extensions SET params = '{\"allowedUserGroups\":[\"6\",\"7\",\"8\"]}'
    WHERE type = 'plugin' AND folder = 'user' AND element = 'token'"
for spec in administrator:Administrator manager:Manager publisher:Publisher editor:Editor author:Author registered:Registered; do
    IFS=: read -r login group <<< "$spec"
    joomla-console user:add --username="$login" --name="$group" --email="$login@example.org" \
        --password="$login-password" --usergroup="$group"
done
joomla-console user:list
# Point the Integration suite at this MariaDB and skip its LDAP tests, as tests/Integration/README.md describes. A
# constant phpunit.xml.dist no longer has is reported, not fatal: like a failing test, it must not stop the scan.
php <<'PHP'
<?php
$settings = ['JTEST_DB_ENGINE' => 'mysqli', 'JTEST_DB_HOST' => 'localhost', 'JTEST_DB_NAME' => 'joomla_test',
             'JTEST_DB_USER' => 'joomla', 'JTEST_DB_PASSWORD' => 'joomla', 'JTEST_LDAP_HOST' => ''];
$xml = new DOMDocument();
$xml->load('/src/phpunit.xml.dist');
$xpath = new DOMXPath($xml);
foreach ($settings as $name => $value) {
    $constants = $xpath->query("/phpunit/php/const[@name='$name']");
    if ($constants->length === 1) {
        $constants->item(0)->setAttribute('value', $value);
    } else {
        echo "WARNING: phpunit.xml.dist has no $name constant; the Integration suite may not reach MariaDB\n";
    }
}
$xml->save('/src/phpunit.xml');
PHP
chown www-data:www-data phpunit.xml
joomla-stop
EOF

# Check, offline, that the site, the administrator, the API and the console work, and that Apache keeps the
# checkout's .git private and runs configuration.php instead of serving it. A failing check fails the build: the
# scanner would get an image it cannot use. A failing test does not: the branch is upmerged daily, and one red test
# must not stop the scan. Each suite's exit code is printed and kept in /var/log/phpunit-results.txt, with its output
# in /var/log/phpunit-<suite>.log.
RUN --network=none <<'EOF'
#!/bin/bash
set -euo pipefail
joomla-start
status() { curl -s -o /dev/null -w '%{http_code}' "$@"; }
curl -fsSL http://127.0.0.1/ | grep 'Joomla OSS Scanner' > /dev/null                      # the installed site, not the installer
curl -fsSL http://127.0.0.1/administrator/index.php | grep 'name="username"' > /dev/null   # the administrator login form
[[ "$(status http://127.0.0.1/api/index.php/v1/content/articles)" =~ ^40[13]$ ]]          # the API refuses anonymous requests
api() { curl -s -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $(joomla-api-token "$1")" "http://127.0.0.1/api/index.php/v1/$2"; }
curl -fsS -H "Authorization: Bearer $(joomla-api-token superuser)" http://127.0.0.1/api/index.php/v1/users | grep '"superuser"' > /dev/null
test "$(api administrator users)" = 200           # Administrators manage users, through the API too,
test "$(api administrator extensions)" = 403      # but not extensions, as in the administrator
test "$(api manager content/articles)" = 200      # Managers manage content,
test "$(api manager users)" = 403                 # not users
test "$(api author content/articles)" = 401       # the site content groups have no API, as on a stock site
test "$(status http://127.0.0.1/installation/index.php)" = 403
test "$(status http://127.0.0.1/libraries/vendor/composer/installed.json)" = 403
test "$(status http://127.0.0.1/phpunit.xml)" = 403
test "$(status http://127.0.0.1/build/build.mjs)" = 403
test "$(status http://127.0.0.1/.git/HEAD)" = 403
# Files a site owner adds at the top level: a robots.txt made from robots.txt.dist is served, as on a release; any other,
# such as a sitemap.xml, is not (see the Apache comment above). Both are removed before the result is checked.
install -o www-data -g www-data -m 0644 robots.txt.dist robots.txt
printf '%s\n' '<?xml version="1.0" encoding="UTF-8"?>' '<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"/>' > sitemap.xml
robots=$(status http://127.0.0.1/robots.txt)
sitemap=$(status http://127.0.0.1/sitemap.xml)
rm -f robots.txt sitemap.xml
echo "Apache, files a site owner adds: /robots.txt $robots (expected 200), /sitemap.xml $sitemap (expected 403)"
test "$robots" = 200
test "$sitemap" = 403
test "$(status http://127.0.0.1/configuration.php)" = 200                                 # configuration.php is executed,
test -z "$(curl -s http://127.0.0.1/configuration.php)"                                    # not served
curl -fsS http://127.0.0.1/LICENSE.txt | cmp - /src/LICENSE.txt
joomla-console cache:clean
suite() {
    # Run a PHPUnit suite, keep its log and state its exit code. Failing tests do not fail the build; a PHPUnit that
    # cannot run at all (126, 127) does, since the image would lack what threat_model.md promises. A log that cannot be
    # saved is reported on its own line and leaves the suite's exit code as PHPUnit returned it.
    local status=(0 0)
    joomla-phpunit --testsuite "$1" 2>&1 | tee "/var/log/phpunit-$1.log" || status=("${PIPESTATUS[@]}")
    local code=${status[0]}
    if [ "${status[1]}" -ne 0 ]; then
        echo "WARNING: PHPUnit $1 suite: its log /var/log/phpunit-$1.log could not be saved (tee exit code ${status[1]})"
    fi
    echo "$1 $code" >> /var/log/phpunit-results.txt
    case $code in
        0) echo "PHPUnit $1 suite: passed (exit code 0)" ;;
        126|127) echo "PHPUnit could not run (exit code $code): see /var/log/phpunit-$1.log" >&2; exit 1 ;;
        *) echo "WARNING: PHPUnit $1 suite: exit code $code; its log is /var/log/phpunit-$1.log" ;;
    esac
}
: > /var/log/phpunit-results.txt
suite Unit
suite Integration
joomla-stop
# Start the scan with an empty error log and mailbox: what is in them now came from the build.
: > /var/log/apache2/error.log
rm -f /var/mail/joomla/*.eml
EOF

# Keep build products out of the checkout's `git status`.
RUN printf '.oss-scanner/\n' >> /src/.git/info/exclude && git status --short

CMD ["bash"]
