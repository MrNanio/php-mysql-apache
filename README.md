### php-mysql-apache containerized
### Drzewo katalogowe projektu
```
/php-mysql-apache/
├── apache
│   ├── Dockerfile
│   └── demo.apache.conf
├── docker-compose.yml
├── php
│   └── Dockerfile
├── mysql
│   └── Dockerfile
└── public_html
    └── index.php
```
### Docker Compose
#### docker-compose.yml
```
version: "3.2"
services:
  php:
    build: 
      context: './php/'
    networks:
      - backend
    volumes:
      - "./public_html/:/var/www/html/"
    container_name: php
  apache:
    build: 
      context: './apache/'
    depends_on:
      - php
      - mysql
    networks:
      - frontend
      - backend
    ports:
      - "6666:80"
    volumes:
      - "./public_html/:/var/www/html/"
    container_name: apache
  mysql:
    build:
      context: './mysql/'
    restart: always
    ports:
      - "3306:3306"
    volumes:
            - data:/var/lib/mysql
    networks:
      - backend
    environment:
      MYSQL_USER: 'USER'
      MYSQL_ALLOW_EMPTY_PASSWORD: "no"
      MYSQL_ROOT_PASSWORD: ''
      MYSQL_PASSWORD: ''
      MYSQL_DATABASE: 'TESTDB'
    container_name: mysql
networks:
  frontend:
  backend:
volumes:
    data:

```
#### Apache
#### apache/Dockerfile
```
FROM httpd:2.4-alpine

RUN apk update; \
    apk upgrade;
    
# Copy apache vhost file to proxy php requests to php-fpm container
COPY demo.apache.conf /usr/local/apache2/conf/demo.apache.conf
RUN echo "Include /usr/local/apache2/conf/demo.apache.conf" \
    >> /usr/local/apache2/conf/httpd.conf


```
#### apache/demo.apache.conf
```
ServerName localhost

LoadModule deflate_module /usr/local/apache2/modules/mod_deflate.so
LoadModule proxy_module /usr/local/apache2/modules/mod_proxy.so
LoadModule proxy_fcgi_module /usr/local/apache2/modules/mod_proxy_fcgi.so

<VirtualHost *:80>
    # Proxy .php requests to port 9000 of the php-fpm container
    ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://php:9000/var/www/html/$1
    DocumentRoot /var/www/html/
    <Directory /var/www/html/>
        DirectoryIndex index.php
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    # Send apache logs to stdout and stderr
    CustomLog /proc/self/fd/1 common
    ErrorLog /proc/self/fd/2
</VirtualHost>
```

### PHP
#### php/Dockerfile
```
FROM php:7.4.3-fpm-alpine
RUN apk update; \
    apk upgrade;

RUN docker-php-ext-install mysqli
```
### MySQL
#### mysql/Dockerfile
```
FROM oraclelinux:7-slim

ARG MYSQL_SERVER_PACKAGE=mysql-community-server-minimal-5.7.32
ARG MYSQL_SHELL_PACKAGE=mysql-shell-8.0.22

# Install server
RUN yum install -y https://repo.mysql.com/mysql-community-minimal-release-el7.rpm \
      https://repo.mysql.com/mysql-community-release-el7.rpm \
  && yum-config-manager --enable mysql57-server-minimal \
  && yum install -y \
      $MYSQL_SERVER_PACKAGE \
      $MYSQL_SHELL_PACKAGE \
      libpwquality \
  && yum clean all \
  && mkdir /docker-entrypoint-initdb.d

VOLUME /var/lib/mysql
EXPOSE 3306 33060
CMD ["mysqld"]
```

