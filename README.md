# CodeIgniter4-Docker-Apache-PHP-MySQL


Now, we have to create “docker-compose.yml”, which we will put in our project folder (in our case “project_folder”). This file should contain, for starters, the following:
```
version: '3'
services:
```
In our docker-compose.yml we will tell docker that we need two, or three, containers – depends on your needs. The first one is the Apache server with PHP, the second will be the database container, and the third, if you want, can be a database client, namely “Adminer”.

### The “db” container

Considering that we will need a MySQL database, let’s start by defining the “db” container in our “services”. So far, the docker-compose.yml file will look like this:

```
version: '3'
services:
    db:
        build:
            context: .
            dockerfile: docker/mysql/Dockerfile
        environment: 
            MYSQL_DATABASE: ${MYSQL_DATABASE}
            MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
        command: --default-authentication-plugin=mysql_native_password
        restart: unless-stopped
        volumes:
            - ./db_data:/usr/data
        ports:
            - 3306:3306
volumes:
    db_data:
```


What did we do here? We’ve told Docker that, in order to create a db container it will need to use the “Dockerfile” file that can be found in “docker/mysql”. We also told it to restart until is manually stopped or until there is a problem with the service. We’ve also declared the local volume that the container should use in order to save the data, and we’ve opened the 3306 port for communication between containers.

I’ve said earlier something about a “Dockerfile”. So, let’s create one in “docker/mysql”. Don’t worry. This will be fast, as it looks like this:

```
FROM mysql

ENV MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
ENV MYSQL_DATABASE=${MYSQL_DATABASE}

#COPY ./docker/mysql/codeigniter_database.sql /docker-entrypoint-initdb.d/init.sql

EXPOSE 3306
```


And, that’s it. We simply told docker to get the latest version of MySQL and expose 3306 port to the “world”. If we want, we can also choose the MySQL version by doing something like: FROM mysql:5.7 . See more here: https://hub.docker.com/_/mysql

We also told it to create a root password that can be retrieved from an environment variable name MYSQL_ROOT_PASSWORD and to create a database whose name is also mentioned in an environment variable named MYSQL_DATABASE.

You may also see a commented line that starts with COPY . If you want to start a project with some default tables, you can at any tyme save the tables in a file (maybe “codeigniter_database.sql“?) and serve it to our container as starting point for the MySQL database.

I mentioned something about environment variables. This environment variables can be saved in an “.env” file that can be saved in our project folder (in my case in the “project_folder”).

So, let’s create one that will contain our first variables:

```
MYSQL_ROOT_PASSWORD=cod31gn1t3
MYSQL_DATABASE=codeigniter_db
```


Nice. Now, going further, let’s install our Apache server container.

## The “web” container

We create the server container by adding the “web” service to our docker-compose.yml file, which now should look like this:

```
version: '3'
services:
    db:
        build:
            context: .
            dockerfile: docker/mysql/Dockerfile
        environment: 
            MYSQL_DATABASE: ${MYSQL_DATABASE}
            MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
        command: --default-authentication-plugin=mysql_native_password
        restart: unless-stopped
        volumes:
            - ./db_data:/usr/data
        ports:
            - 3306:3306
    web:
        build:
            context: .
            dockerfile: docker/apache/Dockerfile
            args:
                uid: ${UID}
        environment:
            - APACHE_RUN_USER=#${UID}
            - APACHE_RUN_GROUP=#${UID}
        restart: unless-stopped
        volumes: 
            - ./public:/var/www/html/public
            - ./apache_log:/var/log/apache2
        ports:
            - 80:80
        depends_on: 
            - db
        links:
            - db
volumes:
    db_data:
    src:
    
```


What did we say in here? We told docker that we have a Dockerfile that we want to be used in order to set up the Apache server container. We want to pass some arguments (args) to the Dockerfile, arguments that will also be taken from our environment variables. We want the container to restarted unless manually stopped and also we want it to use the “src” local directory for its “var/www/html” directory. We will use the 80 port for communicating between our computer and the server and we’ve let the Apache to depend and to be linked to the “db” service.

Now, let’s also look to our other “Dockerfile”, the one that we’ve told Docker to use for Apache and PHP. Let’s create it in “docker/apache” directory and put the following in:


```

FROM php:7.4-apache
RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf
RUN apt-get update
RUN apt-get autoremove
RUN apt-get install -y \
    git \
    zip \
    curl \
    sudo \
    unzip \
    libicu-dev \
    libbz2-dev \
    libpng-dev \
    libjpeg-dev \
    libmcrypt-dev \
    libreadline-dev \
    libfreetype6-dev \
    g++

RUN docker-php-ext-install bz2 intl opcache bcmath calendar pdo_mysql mysqli 


COPY docker/apache/000-default.conf /etc/apache2/sites-available/000-default.conf
RUN a2enmod rewrite headers
RUN mv "$PHP_INI_DIR/php.ini-development" "$PHP_INI_DIR/php.ini"


RUN curl -sS https://getcomposer.org/installer | php
RUN mv composer.phar /usr/local/bin/composer
RUN chmod +x /usr/local/bin/composer
RUN composer self-update

COPY src/ /var/www/html/
#WORKDIR /var/www/html
#RUN composer create-project codeigniter4/appstarter ./

ARG uid
RUN useradd -G www-data,root -u $uid -d /home/devuser devuser
RUN mkdir -p /home/devuser/.composer && \
    chown -R devuser:devuser /home/devuser && \
    chown -R devuser:devuser /var/www/html



EXPOSE 80


```

So… we will use Apache with PHP 7.4, we will name our server localhost, and we start by updating and installing some needed PHP extensions. After that, we establish a new apache config file which we create in “docker/apache” directory.

After this, we install Composer, we copy everything we find from our local “src” directory into the containers “/var/www/html” directory.

And, at the end we set up some read/write rights for our root user that is nemd in our UID environment variable.

Now let’s update the “.env” file with the following variable:

```
UID=1000
```

I said something earlier about the apache config file. So, let’s create a file “000-default.conf” into our “docker/apache” directory and put the following into it:

```
<VirtualHost *:80>
  ServerAdmin webmaster@localhost
  DocumentRoot /var/www/html
  
  <Directory /var/www/html>
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
  </Directory>
</VirtualHost>
```


All done… Unless you also want a MySQL visual client, like Adminer.

## Set up “Adminer” container? Simple…

We just need to add the service to our “docker-compose.yml”. So the file should now look like this:

```
version: '3'
services:
    db:
        build:
            context: .
            dockerfile: docker/mysql/Dockerfile
        environment: 
            MYSQL_DATABASE: ${MYSQL_DATABASE}
            MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
        command: --default-authentication-plugin=mysql_native_password
        restart: unless-stopped
        volumes:
            - ./db_data:/usr/data
        ports:
            - 3306:3306
    web:
        build:
            context: .
            dockerfile: docker/apache/Dockerfile
            args:
                uid: ${UID}
        environment:
            - APACHE_RUN_USER=#${UID}
            - APACHE_RUN_GROUP=#${UID}
        restart: unless-stopped
        volumes: 
            - ./src:/var/www/html
            - ./apache_log:/var/log/apache2
        ports:
            - 80:80
        depends_on: 
            - db
        links:
            - db
    adminer:
        image: adminer
        restart: unless-stopped
        ports:
            - 8080:8080
volumes:
    db_data:
    src:
    
    
```


Now, if we go to http://localhost:8080, we will have access to Adminer.

OK. Done. Now, if we go to our “project_folder” and do a “docker-compose up” in our terminal, all our stack should be built, and we can go to work.

We can test that everything is alright by creating an index.php file in our “public” directory, and we put the following code in it:


```
<?php

$host = 'db';
$user = 'root';
$pass = 'cod31gn1t3';
$conn = new mysqli($host, $user, $pass);

if ($conn->connect_error) {
die("Connection failed: " . $conn->connect_error);
}
echo "Connected to MySQL successfully!";
```


If everything went alright, if we go to http://localhost, we should see that we are connected to MySQL using the mysqli driver. Don’t worry, we’ve also installed the PDO driver, just in case you like it better.







https://avenir.ro/codeigniter-4-using-docker-apache-mysql/?unapproved=6751&moderation-hash=1b161713fa51ef4b06f2adf74d63f360&comment=6751#comment-6751

https://tecadmin.net/install-codeigniter-ubuntu-20-04/
