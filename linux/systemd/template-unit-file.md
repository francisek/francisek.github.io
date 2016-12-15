# What it is

First RTFM : https://www.freedesktop.org/software/systemd/man/systemd.unit.html

Template unit files allow to create a pattern for services definition.

They are loaded in variaous directories depending on the mode systemd is ran (e.g. `/etc/systemd/system`, `/run/systemd/system`, `/usr/lib/systemd/system` directory for system mode).
The filename is `<unit name>@.<unit type>`, ie: `php-fpm@.service` for a unit named `php-fpm` of type `service`.
The content is a text file with sections and options according to documentation.

# Learn by example

1. Create the template file

    Let's create a template to run various php-fpm versions.
    Say that each version is in a subdirectory `/opt/php/X.Y` where `X` is the major and `Y` the minor version.
    The binary is located in subdir `sbin` and is called `php-fpm`.
    
    We'll make a system wide template `/etc/systemd/system/php-fpm@.service` :
    ```bash
    [Unit]
    Description=PHP FPM service
    
    [Service]
    Type=forking
    ExecStartPre=/sbin/runuser -c "/opt/php/%i/sbin/php-fpm --test "
    ExecStart=/sbin/runuser -c "/opt/php/%i/sbin/php-fpm --daemonize --prefix /opt/php/%i --pid /var/run/php-fpm-%i.pid "
    ExecReload=/bin/kill -HUP $MAINPID
    PIDFile=/var/run/php-fpm-%i.pid
    RemainAfterExit=no
    RestartSec=2s
    TimeoutSec=30s
    ```
    
    The template will be called with the version of php-fpm we need.
    
    > - We assume that the search path to php-fpm config files where defined at compilation time.
    > - For php-fpm, the `--pid` parameter seems to be ignored and the value should be set in php-fpm.conf.

1. Register

    After any modification to service files, you should reload the systemd daemon :
    ```bash
    $ sudo systemctl daemon-reload
    ```
    
1. Run service instance

    The the magic operates. Let start the service for version 7.1 :
    ```bash
    $ sudo systemctl start php-fpm@7.1
    ```
    or
    ```bash
    $ sudo service php-fpm@7.1 start
    ```
    
    In the template, every `%i` occurences will be substituted with the part after the `@` and eventually before `.service`.