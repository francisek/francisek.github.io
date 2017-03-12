# What it is

First RTFM : https://www.freedesktop.org/software/systemd/man/systemd.unit.html

Template unit files allow to create a pattern for services definition.

They are loaded in variaous directories depending on the mode systemd is ran (e.g. `/etc/systemd/system`, `/run/systemd/system`, `/usr/lib/systemd/system` directory for system mode).
The filename is `<unit name>@.<unit type>`, ie: `php-fpm@.service` for a unit named `php-fpm` of type `service`.
The content is a text file with sections and options according to documentation.

# Learn by example

## Create the template file

Let's create a template to run various php-fpm versions.
Say that each version is in a subdirectory `/opt/php/X.Y` where `X` is the major and `Y` the minor version.
The binary is located in subdir `sbin` and is called `php-fpm`.

We'll make a system wide template `/etc/systemd/system/php-fpm@.service` :

```bash
[Unit]
Description=PHP FPM service
After=network.target

[Service]
Type=forking
ExecStartPre=/sbin/runuser -c "/opt/php/%i/sbin/php-fpm --test "
ExecStart=/sbin/runuser -c "/opt/php/%i/sbin/php-fpm --daemonize --prefix /opt/php/%i --pid /var/run/php-fpm-%i.pid "
ExecReload=/bin/kill -HUP $MAINPID
PIDFile=/var/run/php-fpm-%i.pid
RemainAfterExit=no
RestartSec=2s
TimeoutSec=30s

[Install]
WantedBy=multi-user.target
```
  
The template will be called with the version of php-fpm we need.
    
  > - We assume that the search path to php-fpm config files where defined at compilation time.
  > - For php-fpm, the `--pid` parameter seems to be ignored and the value should be set in php-fpm.conf.

Explainations

The `Unit` section describes the service.
The `After` entry defines a dependency with the network service.

The `Service` section defines the service:

- The `Type` entry is set to `forking` because the php-fpm process is run in background and forks itself.
- `ExecStartPre` runs before attempting to exec service and checks that configuration is valid.
- `ExecStart` is the command to run the service
- `ExecReload` is the command to reload the service. Here we send a HUP signal to the service PID. We could have send the USR2 signal to only reload configuration.

The `Install` defines how to install the service. This is used to run the service automatically. Here we tell systemd that when enabled, the service will be launched at boot time.

## Register

After any modification to service files, you should reload the systemd daemon:

```bash
$ sudo systemctl daemon-reload
```

## Run service instance

The the magic operates. Let start the service for version 7.1:

```bash
$ sudo systemctl start php-fpm@7.1
```

or

```bash
$ sudo service php-fpm@7.1 start
```

In the template, every `%i` occurences will be substituted with the part after the `@` and eventually before `.service`.

## Automate service loading

If the `Install` section has a `WantedBy` or a `RequiredBy` entry, you can automate the service loading depending another one by simply running:

```bash
$ sudo systemctl enable php-fpm@7.1
```

