# Purpose
Phing Service, build script to expose an unique script to setup system
# Developer environment installation
Simply require fei/phing-service in composer
# Environment configuration information
All projects can use this service to simplify its own build
# Documentation
## Installation
Require fei/phing-service
```bash
composer require fei/phing-service
```
> NB : After tag 2.*

### Phing configuration
#### Build configuration
Add phing.xml script with :
```xml
<project name="[PROJECT-NAME]" default="help" basedir=".">
  <import file="${project.basedir}/vendor/fei/phing-service/setup-system.xml"/>

  <target name="setup-system" description="Execute Deployment, Create DB" depends="setup-system-build"/>
  <target name="local-setup-system" description="Execute Local Deployment" depends="local-setup-system-build"/>
</project>
```
> NB : Targets could be added on this build.xml to take care about project exception

Create settings_local.ini script with minimum on root :
```ini
config.path=[CONFIG PATH]
qaPatern=[CONTAINER PHP NAME]
qaUrlPatern=[QA URL]
qaDatabase=[CONTAINER DB NAME]
qaBeanstalk=[CONTAINER BEANSTALK NAME] (if needed)
```
> NB : After tag 3.* you have to prefix [CONFIG PATH] with ${project.basedir}

Complete this script with your properties with following pattern :
```ini
[NAME OF CONFIG FILE].[YOUR OWN KEY]
```
#### Prepare config files
- Rename Config files .php to .php.dist
- Replace all environment vars to @[NAME OF CONFIG FILE].[YOUR OWN KEY]@

Before
```php
<?php
return [
    new EntityManager('branding', [
        'entities.locations' => ['app/features/Branding/src/Entity'],
        'driver'             => 'pdo_mysql',
        'host'               => 'branding_db',
        'port'               => 3306,
        'user'               => 'branding',
        'password'           => 'branding',
        'dbname'             => 'branding',
        'charset'            => 'UTF8',
        'mapping_types'      => []
    ])
];
```
After
```php
<?php
return [
    new EntityManager('branding', [
        'entities.locations' => ['app/features/Branding/src/Entity'],
        'driver'             => '@doctrine.driver@',
        'host'               => '@doctrine.host@',
        'port'               => 3306,
        'user'               => '@doctrine.user@',
        'password'           => '@doctrine.password@',
        'dbname'             => '@doctrine.dbname@',
        'charset'            => 'UTF8',
        'mapping_types'      => []
    ])
];
```
- Rename Config files tests/*.suite.yml to tests/*.suite.yml.dist (except unit.suite.yml)
- Replace all environment vars to [YOUR OWN KEY] (without @)

Before
```
actor: ApiTester
modules:
    enabled:
        - \Helper\Api
        - REST:
            url: http://[containerPHPName]
            depends: PhpBrowser
        - Filesystem
        - Db
    config:
        Db:
            dsn: 'mysql:host=[containerDBName];dbname=branding'
            user: 'root'
            password: ''
            cleanup: true
            reconnect: true
            populator: "vendor/bin/phing 'create-database'"
            populate: true
gherkin:
    contexts:
        default:
            - WebGuy
            - ApiTester
coverage:
    enabled: false
```

After
```
actor: ApiTester
modules:
    enabled:
        - \Helper\Api
        - REST:
            url: http://qaPattern
            depends: PhpBrowser
        - Filesystem
        - Db
    config:
        Db:
            dsn: 'mysql:host=qaDatabase;dbname=branding'
            user: 'root'
            password: ''
            cleanup: true
            reconnect: true
            populator: "vendor/bin/phing -f phing.xml -propertyfile settings_local.ini 'create-database'"
            populate: true
gherkin:
    contexts:
        default:
            - WebGuy
            - ApiTester
coverage:
    enabled: false
```

## Access and coverage
Add script .htaccess on public path with :
```apache
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} -s [OR]
RewriteCond %{REQUEST_FILENAME} -l [OR]
RewriteCond %{REQUEST_FILENAME} -d
RewriteRule ^.*$ - [L]
RewriteCond %{REQUEST_URI}::$1 ^(/.+)/(.*)::\2$
RewriteRule ^(.*) - [E=BASE:%1]
RewriteRule ^(.*)$ %{ENV:BASE}/index.php [L]
```

Add include c3.php into public/index.php
```php
<?php
if (file_exists(__DIR__ . '/../c3.php')) {
    include __DIR__ . '/../c3.php';
}
```
> NB : You have to create acceptance path too

# Local run
To prepare your environment, you have to run following command :
```
vendor/bin/phing -f phing.xml local-setup-system
```
To prepare your environment for acceptance tests, you have to run following command :
```
vendor/bin/phing -f phing.xml load-properties setup-acceptance
```
# Continuousphp configuration
## On Build settings pipeline fill :
>DocumentRoot => /public
>Phing -> Build file => phing.xml

## On Test settings fill on Codeception :
>Environment variables
>- CONTINUOUSPHP => continuousphp
>- CPHP_SERVICE_APACHE => 2.4.0
>- APP_ENV => dev
>- CPHP_SERVICE_SELENIUM => 3.4.0-chromium

>Phing Target
>- local-setup-system
>- setup-acceptance
>- create-database (in case of acceptance tests)
>- setup-coverage

Overwrite acceptance variable(s)
>- qaPattern => apache24
>- qaDatabase => mysql
>- qaUrlPattern => 127.0.0.1
>- qaBeanstalk => beanstalkd (if needed)

> NB : If you have some trouble with coverage acceptance, please repport on branding-api project to check settings coverage on suites yml

## On Package Settings (for deploy pipeline) :
>Phing Target
>- clean-build

## Addons
Version \>= V4.2.0
### Frontend execution
In case of you have a project >Frontend/backend in the same repository, you can use this target to automaticly build into your backend build without actions of infra.

In your settings ini, you have to add :
```
#Front settings
front.git.url=@github.com/[YOUR FRONTEND PROJECT].git
front.git.version=v1.0.0                    #Frontend version you would like to build
front.workingdir=build                      #Folder where you build your front
front.config.path=src/resources             #Path for your config file
local.front.config.path=src/resources/conf
```

Target to add in continuous at the "Package Settings" step
```
front-setup-config
```

### Connect key generating
If you have to plug connect on your project, simply add the private key in following property
```apacheconfig
connect.key.private=[YOUR PRIVATE KEY]
```