# Select PHP version with sub-domain

In this tutorial we will see how to configure nginx to select a PHP version with a sub-domain prefix.
Let's say we have the domain example.com, we would like serve pages with PHP 7.0 on the example.com domain, and PHP 7.1 on the php7.1.example.com domain.
We will use php-fpm with socket support.

# Requirements

We must have the Php versions installed in a normalized directory structure and use symlinks for minor version accessing.
For example, we can put the version 7.1.0 in the directory `/opt/php/7.1.0` and have a symlink `/opt/php/7.1` pointing to it.
The socket file will be `/var/run/php-fpm-7.1.socket`.
The version 7.0.11 would also be in the `/opt/php/7.0.11` directory with the symlink `/opt/php/7.0` pointing to it.
The socket file will be `/var/run/php-fpm-7.0.socket`.

# Configuring PHP

Each PHP version should have its own proper configuration.
In the pool configuration file, specify the socket file in the `listen` entry.  

# Configuring nginx

nginx loads additionnal configuration with extension `.conf` from the `/etc/nginx/conf.d` directory.
We will create two files in this directory :

- `php-upstream.conf` :
    
    ```
    upstream php-7.0 {
      server unix:/var/run/php-fpm-7.0.socket; 
    }
    
    upstream php-7.1 {
      server unix:/var/run/php-fpm-7.1.socket; 
    }
    ```
    
    This will define a reusable configuration for each php version.
    We just say where to find the socket file.
    
- `per-host-php-version.conf`
    ```
    map $http_host $php_version {
      default "7.1";
      "~^php(?<ver>\d+\.\d+)\.(.*)" $ver;
     }
    ```
    
    - We use the map directive to define the variable $php_version from the $http_host variable. It will be evaluated on demand.
    - On the second line we say that the default value for the $php_version will be `7.1`
    - On the third line, we say that to deduce the value of $php_version from $http_host, we use a regualr expression.
    This regexp will match any url starting with 'php' followed by a numeric value dot a numeric value dot anything.
    The version number will be cach as variable $ver and assigned to $php_version.
    
Then we will create a block (a virtual host in nginx) for our domain. This file will be in the directory `/etc/nginx/sites-available` and simlinked into `/etc/nginx/sites-enabled` under the name `example.com` :
```
server {
    listen 80;
    server_name *.example.com;
    root /var/www;
    index index.php;
    location ~ \.php(/.*|)$ {
        try_files $uri =404;
        fastcgi_pass php-$php_version;
        fastcgi_index index.php;
        include fastcgi.conf;
    }
}
```
    
    We tell nginx to listen on port 80 for the domain `example.com` and its subdomains.
    The served page are located on the server in the `/var/www` directory.
    The default index file used when none is specified is `index.php`.
    Then we say that for any request containing the string `.php/` or ending by `.php`
   
        - nginx will try the file or throw an http error 404
        - then it will pass the request to the php_version-based stream we defined earlier.
     
We just have to reload the nginx service to use the modifications.
     
 
