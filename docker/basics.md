# Docker basics

## Introduction

Docker is half way between virtualisation and native.
The main concepts are :

- docker is based on linux containers [LXC](https://en.wikipedia.org/wiki/LXC) and [cgroups](https://en.wikipedia.org/wiki/Cgroups)
- containers share the same OS kernel and host resources
- container run in a isolated environment
- containers use layered images
- a container should provide at most one service
 
The most common image repository is [dockerhub](https://hub.docker.com/)

## Running an image

An pre-built image can be run from command line :

```bash
$ docker run nginx

```

This runs the nginx image on foreground.
In another terminal, you can list the running containers :

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
5dac3fc1909f        nginx               "nginx -g 'daemon off"   1 minutes ago       Up 1 minutes        80/tcp, 443/tcp     nostalgic_turing
```

Docker has named the container nostalgic_turing, we can see it can provide service on ports 80 (http) and 443 (https).

> The container name changes at each run

But for now we are unable to access thos services because they are isolated.
Stop the image by pressing `CTRL+C` in the terminal running the image or by running :

```bash
$ docker stop nostalgic_turing
```

### Mapping ports

We should open the ports to the world by mapping them. 

```bash
$ docker run -p 81:80 nginx

```

> Options take place between the docker command and the image name

Now the process list looks like :

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                         NAMES
4ece6f66d49f        nginx               "nginx -g 'daemon off"   About a minute ago   Up About a minute   443/tcp, 0.0.0.0:81->80/tcp   distracted_bardeen
```

Now the http service is available on host at port 81. When we browse the url `http://127.0.0.1:81` we can see the default nginx page.
How to put our pages ?

### Mounting volumes

The nginx image documentation on dockerhub says it serves pages from `/usr/share/nginx/html`.
We can create an index file :
```bash
$ mkdir html
$ echo 'Hello world' > html/index.html
```

And then mount the directory `html` into the container :

```bash
$ docker run -p 81:80 -v $(pwd)/html:/usr/share/nginx/html nginx
```

When we browse the url `http://127.0.0.1:81` we can see `Hello world`.

### Naming containers

We can name our container explicitly on command line :

```bash
$ docker run --name docker-nginx -p 81:80 -v $(pwd)/html:/usr/share/nginx/html nginx

```

Now the process list looks like :

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                           NAMES
961036bc76e9        nginx               "nginx -g 'daemon off"   5 seconds ago       Up 4 seconds        443/tcp, 0.0.0.0:81->80/tcp     docker-nginx
```

Each time we have ran `docker run`, docker has created a new container with its specific configuration.
We can re-run those container with the `start command` :

```bash
$ docker start docker-nginx
```

The container runs in background.

### Deleting containers

Now its time to clean up the lot of docker containers we have generated.
List them all :
```bash
$ docker ps -a 
CONTAINER ID        IMAGE                      COMMAND                   CREATED             STATUS                           PORTS                         NAMES
961036bc76e9        nginx                      "nginx -g 'daemon off"    10 minutes ago      Exited (0) 8 minutes ago         443/tcp, 0.0.0.0:81->80/tcp   docker-nginx
4ece6f66d49f        nginx                      "nginx -g 'daemon off"    36 minutes ago      Exited (0) 31 minutes ago                                      distracted_bardeen
5dac3fc1909f        nginx                      "nginx -g 'daemon off"    38 minutes ago      Exited (0) 36 minutes ago                                      nostalgic_turing
1dd3d88204ad        nginx                      "nginx -g 'daemon off"    40 minutes ago      Exited (0) 39 minutes ago                                      jovial_torvalds
```

And delete !

```bash
$ docker rm big_spence docker-nginx distracted_bardeen nostalgic_turing jovial_torvalds
```

## Manage images

### Create an image

An custom image is build upon a definition stored in a text file named `Dockerfile`. 
This provides a way to alter existing image by adding a layer container.

See the [reference of Dockerfile](https://docs.docker.com/engine/reference/builder/)

Sample:

```bash
$ mkdir -p docker-test/docker1/html
$ cd docker-test/docker1
$ echo 'Hello world' > html/index.html
$ cat <<'DOCKERFILE' > Dockerfile
FROM nginx
ADD html /usr/share/nginx/html
DOCKERFILE

```

Then create an image named `docker-nginx-image`:
```docker
$ docker build -t docker-nginx-image .
Sending build context to Docker daemon 3.584 kB
Step 1 : FROM nginx
 ---> 3edcc5de5a79
Step 2 : ADD html /usr/share/nginx/html
 ---> Using cache
 ---> a712dcc3d515
Successfully built a712dcc3d515
```

And run the image :

```bash
$ docker run -p 81:80 docker-nginx-image
```

Our html index is now a part of the image.

### Delete images

List the images :

```bash
$ docker images
docker-nginx-image       latest              a712dcc3d515        10 minutes ago      182.8 MB
nginx                    latest              3edcc5de5a79        11 minutes ago      182.8 MB

```

And delete the docker-nginx-image :

```bash
$ docker rmi docker-nginx-image
```

### Image internal

There are to crucial entries in the `Dockerfile` :

- [ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#/entrypoint) that specifies an executable file to run each time the image is ran.
    The entrypoint executable may :
    
    - handle commands launchable from the host.
    - setup the image on its first execution
    - create users and/or fix permissions
        
    The image will be up until the entrypoint executable runs.
    
- [CMD](https://docs.docker.com/engine/reference/builder/#/cmd) that specifies the default command name to pass to the ENTRYPOINT

Generally the entrypoint execs the main service which will recieve the PID 1.
Inside an image, there is only one default user : `root`.


### Special containers : data containers

A data container does not provide any service. It only holds data.
That means, the main process should be the entrypoint, so generally, the entrypoint script never returns to keep the container alive :

Example of entrypoint.sh for data container :

```bash
#!/bin/bash

set -e

while true; do
    sleep 10000
done
```

or there is no entrypoint at all but a single command that never returns:

Just put it in your Dockerfile:

```
CMD ["/bin/true"]
```

## Compose images

Images can be compose to form an application. For that we use `docker-compose` to manage the images.
It uses a configuration file named `docker-compose.yml` (see [reference](https://docs.docker.com/compose/compose-file)) that describes the services of the application.
The application name is generally the name of the directory hosting the docker-compose.yml file.

### Basic compose

Let's create our previous nginx custom image using docker-compose :

```bash
$ mkdir -p docker-compose-test/docker1/html
$ cd docker-compose-test/docker1
$ echo 'Hello from compose' > html/index.html
$ cat <<'ENDCOMPOSE' > docker-compose.yml
version: '2.1'
services:
    nginx:
        image: nginx
        ports:
            - "81:80"
        volumes:
            - "./html/:/usr/share/nginx/html:ro"
ENDCOMPOSE

```

> The html directory is mounted inside the container as read only.


And run it :

```bash
$ docker-compose up
Recreating 579d1dc4dc78_docker1_nginx_1
Attaching to docker1_nginx_1
```

> Parameters are embeded into the docker-compose file and not needed anymore on command line.

### Service communication

Now let's add a php interpreter to our application.

```bash
$ echo '<?php phpinfo();' > html/index.php
$ cat <<'ENDCOMPOSE' > docker-compose.yml
version: '2.1'
services:
    nginx:
        image: nginx
        ports:
            - "81:80"
        volumes:
            - "./html/:/usr/share/nginx/html:ro"
    internal-php:
        image: php:fpm
        ports:
            - "9000:9000"
ENDCOMPOSE
```

When accessing the url `http://127.0.0.1:81/index.php`, the file is downloaded.
We need to setup nginx to process php files.
For that we will add a new container called internal-php and setup nginx to use it.

```bash
$ cat <<'ENDVHOST' > vhost.conf
server {
    index index.php;
    server_name docker-php.local;
    root /var/www;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        # will use service internal-php on port 9000
        fastcgi_pass internal-php:9000; 
        fastcgi_index index.php;        
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_read_timeout 62;        
    }
}

ENDVHOST
```

and mount the configuration :

```bash
$ cat <<'ENDCOMPOSE' > docker-compose.yml
version: '2.1'
services:
    nginx:
        image: nginx
        ports:
            - "81:80"
        volumes:
            - "./html/:/var/www/"
            - "./vhost.conf:/etc/nginx/conf.d/00_docker-php.conf:ro"
        links:
            - internal-php
    internal-php:
        image: php:fpm
        volumes:
            - "./html/:/var/www/"
ENDCOMPOSE
```

This looks a little bit tricky, but here is the explanations.
The file defines the two services named `nginx` and `php-internal`.
The trick is that both of them must use the same directory structure to process php pages (nginx passes the whole filename to php-fpm).

- The nginx service
 
    - has two mounted directories :

        - the sources are mounted in the `/var/www` directory
        - the http server configuration (pay attention that the nginx image has a default.conf configuration that may be read before yours, I have the habit to prefix the filename with priority)
    
    - needs to communicate with the php image, so we use the `links` directive. Any service inside a docker-compose can be refered by its name, but circular references are prohibited.
    
- The internal-php service

    - has the sources mounted in the same directory as nginx
    - inherits exposed ports from its parent
    
Launch the new application :
```bash
$ docker-compose up --build
```

It works. But as we can see in the docker-compose.yml file, there is a redondant volume entry shared by the two services. This may lead to misconfiguration.
As internal-php has only this volume, we could import it into nginx :

```bash
$ cat <<'ENDCOMPOSE' > docker-compose.yml
version: '2.1'
services:
    nginx:
        image: nginx
        ports:
            - "81:80"
        volumes:
            - "./vhost.conf:/etc/nginx/conf.d/00_docker-php.conf:ro"
        volumes_from:
            - internal-php
        links:
            - internal-php
    internal-php:
        image: php:fpm
        working_dir: /var/www
        volumes:
            - "./html/:/var/www/"
ENDCOMPOSE
```

This looks better. We had to redefine the `working_dir` of internal-php because the php-fpm image defines it as /var/www/html which does not exist anymore.

### Be structurated

Our current directory structure is too flat :

```
.
├── docker-compose.yml
│   └── html
│       ├── index.html
│       └──index.php
└── vhost.conf
```

We should make it more readable :
 
```bash
$ mkdir -p nginx/conf.d sources
$ mv html sources/www
$ mv vhost.conf nginx/conf.d/
```

We have to reflect the changes to the docker-compose.yml :

```bash
cat <<'ENDCOMPOSE' > docker-compose.yml
version: '2.1'
services:
    nginx:
        image: nginx
        ports:
            - "81:80"
        volumes:
            - "./nginx/conf.d/vhost.conf:/etc/nginx/conf.d/00_docker-php.conf:ro"
        volumes_from:
            - internal-php
        links:
            - internal-php
    internal-php:
        image: php:fpm
        working_dir: /var/www
        volumes:
            - "./sources/www/:/var/www/"
ENDCOMPOSE
```

> Its a good practice to have a first level directory structure corresponding to service names. 

### Create a data container

Our application could be better : imagine we add some volumes to the internal-php service. They would be exposed to nginx, that may lead to problems !
We can isolate our source code into a data container.

We must define a custom image for the data container with a Dockerfile :

```bash
$ cat <<'ENDDOCKERFILE' > sources/Dockerfile
FROM alpine
RUN mkdir -p /var/www > /dev/null
CMD ["true"]
ENDDOCKERFILE
```

```bash
$ cat <<'ENDCOMPOSE' > docker-compose.yml
version: '2.1'
services:
    nginx:
        image: nginx
        ports:
            - "81:80"
        volumes:
            - "./nginx/conf.d/vhost.conf:/etc/nginx/conf.d/00_docker-php.conf:ro"
        volumes_from:
            - "sources"
        links:
            - "internal-php"
    internal-php:
        image: php:fpm
        working_dir: /var/www
        volumes_from:
            - "sources"
    sources:
        build: ./sources
        volumes:            
            - "./sources/www:/var/www"

ENDCOMPOSE
```

We tell docker-compose to build the sources container from the Dockerfile located in the sources directory.
We now create that file :

```bash
$ cat <<'ENDDOCKERFILE' > sources/Dockerfile
FROM alpine
RUN mkdir -p /var/www > /dev/null
CMD ["true"]
ENDDOCKERFILE
```

This works well. The local `sources/www` is mounted inside the `sources` data container at `/var/www`.
Any changes into `sources/www` is immediately reflected into the data container.

### Handle rights 
 
We know add a new php file that will write a file into sources :

```bash
$ cat <<'ENDPHP' > sources/www/write.php
<?php
printf('Creating a file into %s:', getcwd());
file_put_contents(getcwd() . '/test', 'Hello world');
echo 'Done';
ENDPHP
```

When accessing to the page we have the following output :

    > Creating a file into /var/www:
    > Warning: file_put_contents(/var/www/test): failed to open stream: Permission denied in /var/www/write.php on line 3
    > Done

As the documentation says, the mounting of a directory does not affect the local file rights : the local rights at create time are applied. 
That means that the permissions (flags, owner and group) defined on the host are applied as is inside the container.
Say the current user has a UID of 1000 and a GID of 1000, the files mounted inside the container will have the same.
When inside a container, a process requests access to a mounted volume, that's the process owner which is used.
In our case, the php-fpm is trying to write a file. Let's see who's its owner :

```bash
$ docker-compose exec internal-php ps aux | grep php
```

The result is :
```
root         1  0.0  0.1 114740 17456 ?        Ss   10:17   0:00 php-fpm: master
www-data     7  0.0  0.1 114804 10656 ?        S    10:17   0:00 php-fpm: pool w
www-data     8  0.0  0.1 114804 10656 ?        S    10:17   0:00 php-fpm: pool w
```

That's the user `www-data` who attempted to create the file.
At this point we have many options to solve the problem:

- we could chmod localy all the sources to allow read/write to everybody.
  This would be a bad solution because:
  
    - That would be a weird security breach on the host
    - Created nodes would be owned by container user www-data that may not exist on host or even worser may be mapped to another user
     
- we could change the uid/gid of user www-data to 1000
  That would work but there are drawbacks :
  
    - it is always a bad idea to hardcode parameters
    - its a bad practice to modify existing user created by a base image, because we do not know exactly what the image does with it.
      
- we could create a proper user to run the php-fpm pool and map it to the gid/uid of the host user.
  Could be nice but what if the host source directory is mapped to another user ?

- we could create a proper user to run the php-fpm pool and map it to a runtime-defined uid/gid.

The fourth solution sounds pretty nice.

To set up this solution, we have to :

- find a way to pass the user's UID/GID
- alter the base image php:fpm, so we need to layer it with a Dockerfile which creates our user
- setup the php pool to use our user

So we prepare the directory structure to host the new files:

```bash
$ mkdir -p internal-php/pool.d
```

### Pass arguments with environment

Arguments can be passed to docker at build-time or at run-time. 
For build-time arguments, refer to the [ARG](https://docs.docker.com/engine/reference/builder/#arg) instruction of Dockerfile and the [args](https://docs.docker.com/compose/compose-file/#args) instruction of Docker-compose.
The arguments are defined for the `build` command with the `--build-arg` flag. 

We will use runtime arguments because that gives us the ability to change and reflect host privileges.

Let's create the php-fpm Dockerfile that will create our custom host mapped user: 

```bash
$ cat <<'ENDDOCKERFILE' > internal-php/Dockerfile
FROM php:fpm
RUN groupadd -r hostgroup && useradd -r -g hostgroup hostuser
# copy the entrypoint
COPY ./entrypoint.sh /entrypoint.sh
# set it executable
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["php-fpm"]
ENDDOCKERFILE
```

> In Dockerfile, relative paths are always relative to the Dockerfile directory 

Then we create an entrypoint that will alter the hostuser each time the image is ran:

```bash
$ cat <<'ENDENTRYPOINT' > internal-php/entrypoint.sh
#!/bin/bash

# exit if a command exists with non-zero status
set -e

HOST_USER_UID=${HOST_USER_UID}
HOST_USER_GID=${HOST_USER_GID}

# Ensure we have at least UID or GID 
if [ -z "$HOST_USER_UID" -a -z "$HOST_USER_GID" ]; then
    echo "Either HOST_USER_UID or HOST_USER_GID should be specified"
    exit 1
fi

# if command starts with an option, prepend php-fpm
if [ "${1:0:1}" = '-' ]; then
        set -- php-fpm "$@"
fi

if [ -n "$HOST_USER_UID" ]; then
    echo "Setting Pool owner to UID ${HOST_USER_UID}"
    usermod --uid $HOST_USER_UID --non-unique hostuser
fi

if [ -n "$HOST_USER_GID" ]; then
    echo "Setting Pool owner to GID ${HOST_USER_GID}"
    groupmod --gid $HOST_USER_GID --non-unique hostgroup
fi

exec "$@"
ENDENTRYPOINT
```

We can then define the newly created user in the php-fpm pool:

```bash
$ cat <<'ENDPOOL' > internal-php/pool.d/www.pool.conf
[global]
daemonize = no


[www]
user = hostuser
group = hostgroup
listen = [::]:9000
listen.owner = hostuser
listen.group = hostgroup

pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
ENDPOOL
```

> - We set a global entry to tell the main process not to daemonize. (If not, the process would be sent to background and the entrypoint would end).
> - In the php:fpm image, only `.conf` files are used to define pools. They are red in alphabetical order.

Now we adjust our docker-compose.yml file to define our two environment variables :

```bash
$ cat <<'ENDCOMPOSE' > docker-compose.yml
version: '2.1'
services:
    nginx:
        image: nginx
        ports:
            - "81:80"
        volumes:
            - "./nginx/conf.d/vhost.conf:/etc/nginx/conf.d/00_docker-php.conf:ro"
        volumes_from:
            - "sources"
        links:
            - "internal-php"
    internal-php:
        build: ./internal-php
        working_dir: /var/www
        volumes:
            - "./internal-php/pool.d/:/usr/local/etc/php-fpm.d/"
        volumes_from:
            - "sources"
        environment:
            - "HOST_USER_UID=${HOST_USER_UID:-1000}"
            - "HOST_USER_GID=${HOST_USER_GID:-1000}"
    sources:
        build: ./sources
        volumes:            
            - "./sources/www:/var/www"
ENDCOMPOSE
```

> We added an environment statement with default value (supported since 2.1 format of docker-compose.yml)

After rebuilding the application, opening the page https://localhost:81/write.php should work fine. The file should be readable from the host.

## Define custom application options

### Define environment

When there is a lot of environment variables to setup the application, exporting them could be extremly boring.
You can put their definition in a `.env` file that will be automagically be red by compose.

Let say the compose files are owned by the www-data user with UID 33 and GID 33 :
```bash
$ cat <<'ENDENV' > .env
HOST_USER_UID=33
HOST_USER_GID=33
ENDENV
```

### Define docker-compose.yml file override

By default Compose may read two files : `docker-compose.yml` and an optional `docker-compose.override.yml`. 
We could for example create the override file to extend services (additional port mapping or volumes definition, replacing env_file).

> - The docker-compose.override.yml must define the same version as in the docker-compose.yml
> - Some values (like `ports` and `expose`) are concatenated and some (like `environment`, `volumes`) are merged.

### Define per environment configuration

You can create service overrides dedicated to prod or dev environments.
See [Different environments](https://docs.docker.com/compose/extends/#/different-environments) for examples.
