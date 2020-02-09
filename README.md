[![StyleCI](https://github.styleci.io/repos/179174116/shield?branch=master)](https://github.styleci.io/repos/179174116)


## Table of contents

* [System Requirements](#system-requirements)
  * [Optional](#optional)
* [Quick Install](#quick-install)
* [Application Structure](#application-structure)
* [Downloading & Compiling](#downloading--compiling)
* [Creating the Jail](#creating-the-jail)
* [Enabling the Worker](#enabling-the-worker)
  * [How it Works](#how-it-works)
* [Example Deployment](#example-deployment)


## System Requirements

- build-essential 
- pkg-config
- libcurl4-openssl-dev
- libxml2-dev
- libtidy-dev

### Optional

- bison
- re2c

## Quick install

Install PHP:

```bash
$ php cli/install --version="<(string)version>"
```

Create jail:
```bash
$ sudo php cli/build --jail="<(string)jailpath>" --version="<(string)version>"
```

## Application Structure
```
.
├── config
│   ├── .env                # Environment variables
│   ├── assets.json         # Front-end assets to compile
│   ├── bootstrap.php       # Application bootstrapper
│   ├── controllers.php     # Place to register application controllers
│   ├── dependencies.php    # Place to register application dependencies
│   └── routes.php          # Place to register application routes
├── node_modules            # Reserved for Yarn
├── public                  # Entry, web and cache files
├── resources               # Application resources
│   ├── assets              # Raw, un-compiled assets such as media, SASS and JavaScript
│   ├── views               # View templates (twig)
├── src                     # PHP source code (The App namespace)
│   ├── Console             # Console commands
│   ├── Frontend            # Configuration files
│       ├── Controllers     # Frontend controllers
├── vendor                  # Reserved for Composer
├── composer.json           # Composer dependencies
├── gulpfile.esm.js         # Gulp configuration
├── LICENSE                 # The license
├── package.json            # Yarn dependencies
└── README.md               # This file
```

## Downloading & Compiling

```php
<?php

use Versyx\Codepad\Console\Compiler;
use Versyx\Codepad\Console\Downloader;

require __DIR__ . '/../config/bootstrap.php';

if(!isset($argv[1])) {
    die("You must specify a PHP version.");
}

$short = 'v:';
$long  = ['version:'];
$opts  = getopt($short, $long);

run(
    $app['downloader'],
    $app['compiler'],
    $opts['version'] ?? $opts['v']
);

function run(Downloader $downloader, Compiler $compiler, string $version)
{
    try {
        $php = $downloader->setVersion($version)->download();
        $compiler->compile($php->getVersion(), $php->getTarget());
    } catch (\Exception $e) {
        echo $e->getMessage();
    }
}
```

## Creating the Jail

```php
<?php

use Versyx\Codepad\Console\Jailer;

require __DIR__ . '/../config/bootstrap.php';

if(!isset($argv[1])) {
    die('You must specify a PHP version.' . PHP_EOL);
}

$short = 'j::v:';
$long  = ['jail::', 'version:', 'first-run::'];
$opts  = getopt($short, $long);

run($app['jailer'], $opts);

function run(Jailer $jailer, array $opts)
{
    if(!$jail = env("JAIL_ROOT")) {
        $jail = $opts['jail'] ?? $opts['j'];
    }

    if(!$version = env("JAIL_PHP_VERSION")) {
        $version = $opts['version'] ?? $opts['v'];
    }

    if(env("JAIL_DEVICES")) {
        $jailer->setDevices(explode(',', env("JAIL_DEVICES")));
    } else {
        $jailer->setDevices(['bin','dev','etc','lib','lib64','usr']);
    }

    if($jail) {
        $jailer->setRoot($jail);
    } else {
        echo 'You must set a root path.' . PHP_EOL;
        exit;
    }

    if($version) {
        $php = '/php-' . $version;

        $jailer->buildJail($version);

        if(isset($opts['first-run'])) {
            $jailer->setPermissions(0711);
            $jailer->setOwnership('root', 'root');
            $jailer->mountAll();
        }
        
        $jailer->mkJailDir($php);
        $jailer->mount('/tmp' . $php, $jailer->getRoot() . $php, 'bind', 'ro');
    } else {
        echo 'You must set a version.' . PHP_EOL;
        exit;
    }
}
```

## Enabling the Worker

You'll need to allow www-data to run the worker script as a privileged user, add entries for each compiled version to
 `/etc/sudoers` like so:


ubuntu:
```
www-data ALL =(ALL) NOPASSWD: /opt/phpjail/php-7.3.6/bin/php /var/www/codepad/public/http/worker.php 7.3.6
www-data ALL =(ALL) NOPASSWD: /opt/phpjail/php-7.0.33/bin/php /var/www/codepad/public/http/worker.php 7.0.33
```

centos:
```
apache ALL =(ALL) NOPASSWD: /opt/phpjail/php-7.3.6/bin/php /var/www/codepad/public/http/worker.php 7.3.6
apache ALL =(ALL) NOPASSWD: /opt/phpjail/php-7.0.33/bin/php /var/www/codepad/public/http/worker.php 7.0.33
```

This will restrict `www-data`'s|`apache`'s sudo privileges to only running the worker.

### How it Works

The PHP code and version is base64 encoded and submitted to `http/manager.php`, the manager then 
base64 decodes the data and runs a check on the code input against disabled functions, if the check
comes back clean, a new process is created with stream resources:

```php
$proc = proc_open("sudo /opt/phpjail/php-$ver/bin/php /var/www/" . env("APP_NAME") . "/public/http/worker.php $ver", [
    0 => ["pipe", "rb"],
    1 => ["pipe", "wb"],
    2 => ["pipe", "wb"]
], $pipes);
```

The PHP code is passed to `http/worker.php` from the manager via STDIN, the worker then creates a temporary file in
`/opt/phpjail`, sets its permissions to `0444` and then executes the file using the selected PHP version
instance, which is chrooted to `/opt/phpjail` as user `nobody`. If the code takes longer than five seconds to execute, 
the process will terminate.

```php
$starttime = microtime(true);
$unused = [];
$ph = proc_open('chroot --userspec=nobody /opt/phpjail /php-' . $argv[1] .'/bin/php ' . escapeshellarg(basename($file)), $unused, $unused);
$terminated = false;
while (($status = proc_get_status($ph)) ['running']) {
    usleep(100 * 1000);
    if (!$terminated && microtime(true) - $starttime > MAX_RUNTIME_SECONDS) {
        $terminated = true;
        echo 'max runtime reached (' . MAX_RUNTIME_SECONDS . ' seconds), terminating...';
        pKill($status['pid']);
    }
}

proc_close($ph);
```

## Example Deployment

![example deployment](resources/assets/img/codepad.png)
