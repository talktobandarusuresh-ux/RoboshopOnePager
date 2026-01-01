# Roboshop 3 Tier Architecture

![roboshop](https://github.com/user-attachments/assets/dfa53a5b-5367-46ca-bc8a-8e2560f3fdea)

# 01-MongoDB

# 1 - create file by using **`vim /etc/yum.repos.d/mongo.repo`**
**Developer has shared the version information as MongoDB-7.x**
Setup the MongoDB repo file 
``` shell title=/etc/yum.repos.d/mongo.repo
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/7.0/x86_64/
enabled=1
gpgcheck=0
```
# 2-Install MongoDB 

```shell 
dnf install mongodb-org -y 
```
# 3-Start & Enable MongoDB Service 

```shell 
systemctl enable mongod 
systemctl start mongod 
```

# 4-Edit file by using **`vim /etc/mongod.conf`**
Note: Usually MongoDB opens the port only to `localhost(127.0.0.1)`, meaning this service can be accessed by the application that is hosted on this server only. However, we need to access this service to be accessed by another server, So we need to change the config accordingly.
Update listen address from 127.0.0.1 to 0.0.0.0 in `/etc/mongod.conf`
# 5-Restart the service to make the changes effected.
```shell 
systemctl restart mongod
```


# 02-Catalogue

Catalogue is a microservice that is responsible for serving the list of items that displays in roboshop application.

**Developer has chosen NodeJs, Check with developer which version of NodeJS is needed.**
**Developer has set a context that it can work with NodeJS >20**

Install NodeJS, By default NodeJS 16 is available, We would like to enable 20 version and install list.

**You can list modules by using `dnf module list nodejs`**

Disable current module & Enable required module
```shell 
dnf module disable nodejs -y
```

```shell
dnf module enable nodejs:20 -y
```

Install NodeJS 
```shell 
dnf install nodejs -y
```

Configure the application.

Our application developed by the developer of our org and it is not having any RPM software just like other softwares. So we need to configure every step manually

We already discussed in Linux basics section that applications should run as nonroot user.

Add application User

```shell 
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
```

User **roboshop** is a function / daemon user to run the application. Apart from that we dont use this user to login to server. Also, username **roboshop** has been picked because it more suits to our project name. We keep application in one standard location. This is a usual practice that runs in the organization.
Lets setup an app directory. 

```shell
mkdir /app 
```

Download the application code to created app directory. 

```shell
curl -o /tmp/catalogue.zip https://roboshop-artifacts.s3.amazonaws.com/catalogue-v3.zip 
cd /app 
unzip /tmp/catalogue.zip
```

Every application is developed by development team will have some common softwares that they use as libraries. This application also have the same way of defined dependencies in the application configuration. Lets download the dependencies. 

```shell 
cd /app 
npm install 
```

We need to setup a new service in **systemd** so `systemctl` can manage this service. We already discussed in linux basics that advantages of systemctl managing services, Hence we are taking that approach. Which is also a standard way in the OS. 
# Setup SystemD Catalogue Service 

```unit file (systemd) title=/etc/systemd/system/catalogue.service
[Unit]
Description = Catalogue Service

[Service]
User=roboshop
Environment=MONGO=true
// highlight-start
Environment=MONGO_URL="mongodb://<MONGODB-SERVER-IPADDRESS>:27017/catalogue"
// highlight-end
ExecStart=/bin/node /app/server.js
SyslogIdentifier=catalogue

[Install]
WantedBy=multi-user.target
```

Hint! You can create file by using **`vim /etc/systemd/system/catalogue.service`**

Ensure you replace `<MONGODB-SERVER-IPADDRESS>` with IP address

Load the service.

```shell 
systemctl daemon-reload
```

This above command is because we added a new service, We are telling systemd to reload so it will detect new service.

Enable and Start the service.

```shell 
systemctl enable catalogue 
systemctl start catalogue
```

For the application to work fully functional we need to load schema to the Database. Additonally we are going to have master data to be seeded for the applications to work. Here in this case it is list of products we want to sale.
Schemas are usually part of application code and developer will provide them as part of development.
Master data are usually provided by business operations team.
To load schema / master data we need to install mongodb client and then we can load it.
To have mongo client installed we have to setup MongoDB repo and install mongodb-client. You can create file using

```
vim /etc/yum.repos.d/mongo.repo
```

``` shell
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/7.0/x86_64/
enabled=1
gpgcheck=0
```

```shell 
dnf install mongodb-mongosh -y
```

Load Master Data of the List of products we want to sell and their quantity information also there in the same master data. 

```
mongosh --host MONGODB-SERVER-IPADDRESS </app/db/master-data.js
```

**NOTE:**
You need to update catalogue server ip address in frontend configuration. 
Configuration file is `/etc/nginx/nginx.conf` 

Use below commands to check data is loaded into mongodb or not
Connect to MongoDB
```
mongosh --host MONGODB-SERVER-IPADDRESS
```

Show databases, Use database, Show Collections, Get the items in collection
```
show dbs
```
```
use catalogue
```
```
show collections
```
```
db.products.find()
```

# 03-Frontend

The frontend is the service in RoboShop to serve the web content over Nginx. This will have the webframe for the web application.
This is a static content and to serve static content we need a web server. This server
Developer has chosen Nginx as a web server and thus we will install Nginx Web Server. 

**You can list modules by using**

`dnf module list nginx`

Install Nginx 
```shell
dnf module disable nginx -y
dnf module enable nginx:1.24 -y
dnf install nginx -y
```

Start & Enable Nginx service 
```shell
systemctl enable nginx 
systemctl start nginx 
```

**Try to access the service once over the browser and ensure you get some default content.**
Remove the default content that web server is serving. 
```shell
rm -rf /usr/share/nginx/html/* 
```

Download the frontend content
```shell
curl -o /tmp/frontend.zip https://roboshop-artifacts.s3.amazonaws.com/frontend-v3.zip
```

Extract the frontend content.
```shell
cd /usr/share/nginx/html 
unzip /tmp/frontend.zip
```

**Try to access the nginx service once more over the browser and ensure you get roboshop content.**
Create Nginx Reverse Proxy Configuration to reach backend services.
```shell 
vim /etc/nginx/nginx.conf
```

Add the following content 
```nginx configuration title=/etc/nginx/nginx.conf 
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        root         /usr/share/nginx/html;

        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }

        location /images/ {
          expires 5s;
          root   /usr/share/nginx/html;
          try_files $uri /images/placeholder.jpg;
        }
        location /api/catalogue/ { proxy_pass http://localhost:8080/; }
        location /api/user/ { proxy_pass http://localhost:8080/; }
        location /api/cart/ { proxy_pass http://localhost:8080/; }
        location /api/shipping/ { proxy_pass http://localhost:8080/; }
        location /api/payment/ { proxy_pass http://localhost:8080/; }

        location /health {
          stub_status on;
          access_log off;
        }

    }
}
```

**NOTE:**
Ensure you replace the `localhost` with the actual ip address of those component server. Word `localhost` is just used to avoid the failures on the Nginx Server.

Restart Nginx Service to load the changes of the configuration.

```shell 
systemctl restart nginx 
```

# 04-Redis

Redis (REmote DIctionary Server) is an open-source, in-memory data store used as a database, cache. It is known for ultra-fast performance due to its in-memory(stores in RAM) architecture. Uses a simple key-value model but supports complex data types.

**Versions of the DB Software you will get context from the developer, Meaning we need to check with developer.**

Install redis, By default redis 6 is available, We would like to enable 7 version and install list.
```shell 
dnf module disable redis -y
dnf module enable redis:7 -y
```

Install Redis 
```shell
dnf install redis -y 
```

Usually Redis opens the port only to `localhost(127.0.0.1)`, meaning this service can be accessed by the application that is hosted on this server only. However, we need to access this service to be accessed by another server, So we need to change the config accordingly.
Update listen address from `127.0.0.1` to `0.0.0.0` in   `/etc/redis/redis.conf`
Update `protected-mode` from `yes` to `no` in   `/etc/redis/redis.conf`
You can edit file by using **`vim /etc/redis/redis.conf`**

Start & Enable Redis Service 
```shell 
systemctl enable redis 
systemctl start redis 
```
# 05-User

User is a microservice that is responsible  for User Logins and Registrations Service in RobotShop e-commerce portal.

**Developer has chosen NodeJs, Check with developer which version of NodeJS is needed.**
**Developer has set a context that it can work with NodeJS >20**
Install NodeJS, By default NodeJS 16 is available, We would like to enable 20 version and install list.
**You can list modules by using `dnf module list`**
```shell 
dnf module disable nodejs -y
dnf module enable nodejs:20 -y
```
Install NodeJS 
```shell 
dnf install nodejs -y
```
Configure the application. Here
Our application developed by the user is not having any RPM software just like other softwares. So we need to configure every step manually
We already discussed in Linux basics section that applications should run as nonroot user.
Add application User
```shell 
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
```

info 
User **roboshop** is a function / daemon user to run the application. Apart from that we dont use this user to login to server.
Also, username **roboshop** has been picked because it more suits to our project name.
We keep application in one standard location. This is a usual practice that runs in the organization.
Lets setup an app directory. 
```shell
mkdir /app 
```
Download the application code to created app directory. 
```shell
curl -L -o /tmp/user.zip https://roboshop-artifacts.s3.amazonaws.com/user-v3.zip 
cd /app 
unzip /tmp/user.zip
```
Every application is developed by development team will have some common softwares that they use as libraries. This application also have the same way of defined dependencies in the application configuration.
Lets download the dependencies. 
```shell 
cd /app 
npm install 
```
We need to setup a new service in **systemd** so `systemctl` can manage this service
We already discussed in linux basics that advantages of systemctl managing services, Hence we are taking that approach. Which is also a standard way in the OS. 
Setup SystemD User Service 

```unit file (systemd) title=/etc/systemd/system/user.service
[Unit]
Description = User Service
[Service]
User=roboshop
Environment=MONGO=true
// highlight-start
Environment=REDIS_URL='redis://<REDIS-IP-ADDRESS>:6379'
Environment=MONGO_URL="mongodb://<MONGODB-SERVER-IP-ADDRESS>:27017/users"
// highlight-end
ExecStart=/bin/node /app/server.js
SyslogIdentifier=user

[Install]
WantedBy=multi-user.target
```

You can create file by using **`vim /etc/systemd/system/user.service`**
Load the service.
```shell 
systemctl daemon-reload
```

This above command is because we added a new service, We are telling systemd to reload so it will detect new service.
enable and Start the service.

```shell 
systemctl enable user 
systemctl start user
```
# 06-Cart

Cart is a microservice that is responsible for Cart Service in RobotShop e-commerce portal.

**Developer has chosen NodeJs, Check with developer which version of NodeJS is needed.**
**Developer has set a context that it can work with NodeJS >20**
Install NodeJS, By default NodeJS 16 is available, We would like to enable 20 version and install list.
**You can list modules by using `dnf module list`**
```shell 
dnf module disable nodejs -y
dnf module enable nodejs:20 -y
```
Install NodeJS 
```shell 
dnf install nodejs -y
```
Configure the application. Here
Our application developed by the own developer is not having any RPM software just like other softwares. So we need to configure every step manually
caution 
We already discussed in Linux basics section that applications should run as nonroot user.

Add application User
```shell 
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
```
 
User **roboshop** is a function / daemon user to run the application. Apart from that we dont use this user to login to server.
Also, username **roboshop** has been picked because it more suits to our project name.
We keep application in one standard location. This is a usual practice that runs in the organization.
Lets setup an app directory. 

```shell
mkdir /app 
```

Download the application code to created app directory. 
```shell
curl -L -o /tmp/cart.zip https://roboshop-artifacts.s3.amazonaws.com/cart-v3.zip
cd /app 
unzip /tmp/cart.zip
```

Every application is developed by development team will have some common softwares that they use as libraries. This application also have the same way of defined dependencies in the application configuration.
Lets download the dependencies. 
```shell 
cd /app 
npm install 
```
We need to setup a new service in **systemd** so `systemctl` can manage this service
We already discussed in linux basics that advantages of systemctl managing services, Hence we are taking that approach. Which is also a standard way in the OS. 
Setup SystemD Cart Service 

```unit file (systemd) title=/etc/systemd/system/cart.service
[Unit]
Description = Cart Service
[Service]
User=roboshop
// highlight-start
Environment=REDIS_HOST=<REDIS-SERVER-IP>
Environment=CATALOGUE_HOST=<CATALOGUE-SERVER-IP>
Environment=CATALOGUE_PORT=8080
// highlight-end
ExecStart=/bin/node /app/server.js
SyslogIdentifier=cart

[Install]
WantedBy=multi-user.target
```
 
You can create file by using **`vim /etc/systemd/system/cart.service`**
Load the service.
```shell 
systemctl daemon-reload
```
This above command is because we added a new service, We are telling systemd to reload so it will detect new service.
enable and Start the service.
```shell 
systemctl enable cart 
systemctl start cart
```

# 07-MySQL 

Developer has chosen the database MySQL. Hence, we are trying to install it up and configure it.

**Versions of the DB Software you will get context from the developer, Meaning we need to check with developer.**
**Developer has shared the version information as MySQL-8.x**

Install MySQL Server 
```shell 
dnf install mysql-server -y
```

Start MySQL Service 
```shell 
systemctl enable mysqld
systemctl start mysqld  
```

Next, We need to change the default root password in order to start using the database service. Use password **`RoboShop@1`** or any other as per your choice. 
```shell
mysql_secure_installation --set-root-pass RoboShop@1
```

# 08-Shipping

Shipping service is responsible for finding the distance of the package to be shipped and calculate the price based on that.
Shipping service is written in Java, Hence we need to install Java.
Maven is a Java Packaging software, Hence we are going to install **`maven`**, This indeed takes care of java installation. 
Developer has chosen Maven, Check with developer which version of Maven is needed.
Here for our requirement java >= 1.8 & maven >=3.5 should work.
```shell 
dnf install maven -y
```
Configure the application.
Our application developed by the developer of our org and it is not having any RPM software just like other softwares. So we need to configure every step manually
We already discussed in Linux basics section that applications should run as nonroot user.
Add application User
```shell 
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
```
User **roboshop** is a function / daemon user to run the application. Apart from that we dont use this user to login to server.
Also, username **roboshop** has been picked because it more suits to our project name.
We keep application in one standard location. This is a usual practice that runs in the organization.
Lets setup an app directory.

```shell
mkdir /app 
```
Download the application code to created app directory.
```shell
curl -L -o /tmp/shipping.zip https://roboshop-artifacts.s3.amazonaws.com/shipping-v3.zip 
cd /app 
unzip /tmp/shipping.zip
```
Every application is developed by development team will have some common softwares that they use as libraries. This application also have the same way of defined dependencies in the application configuration.
Lets download the dependencies & build the application

```shell 
cd /app 
mvn clean package 
mv target/shipping-1.0.jar shipping.jar 
```
We need to setup a new service in **systemd** so `systemctl` can manage this service
We already discussed in linux basics that advantages of systemctl managing services, Hence we are taking that approach. Which is also a standard way in the OS.
Setup SystemD Shipping Service
```unit file (systemd) title=/etc/systemd/system/shipping.service
[Unit]
Description=Shipping Service

[Service]
User=roboshop
// highlight-start
Environment=CART_ENDPOINT=<CART-SERVER-IPADDRESS>:8080
Environment=DB_HOST=<MYSQL-SERVER-IPADDRESS>
// highlight-end
ExecStart=/bin/java -jar /app/shipping.jar
SyslogIdentifier=shipping

[Install]
WantedBy=multi-user.target

```

Hint! You can create file by using **`vim /etc/systemd/system/shipping.service`**
Load the service.
```shell 
systemctl daemon-reload
```

This above command is because we added a new service, We are telling systemd to reload so it will detect new service.
Start the service.
```shell 
systemctl enable shipping 
systemctl start shipping
```

For this application to work fully functional we need to load schema to the Database.
Schemas are usually part of application code and developer will provide them as part of development.
We need to load the schema. To load schema we need to install mysql client.
To have it installed we can use
```shell
dnf install mysql -y 
```
Load Schema, Schema in database is the structure to it like what tables to be created and their necessary application layouts.
```shell 
mysql -h <MYSQL-SERVER-IPADDRESS> -uroot -pRoboShop@1 < /app/db/schema.sql
```
Create app user, MySQL expects a password authentication, Hence we need to create the user in mysql database for shipping app to connect.
```shell 
mysql -h <MYSQL-SERVER-IPADDRESS> -uroot -pRoboShop@1 < /app/db/app-user.sql 
```
Load Master Data, This includes the data of all the countries and their cities with distance to those cities.
```shell 
mysql -h <MYSQL-SERVER-IPADDRESS> -uroot -pRoboShop@1 < /app/db/master-data.sql
```
This service needs a restart because it is dependent on schema, After loading schema only it will work as expected, Hence we are restarting this service. This
```shell 
systemctl restart shipping
```
# 09-RabbitMQ 

RabbitMQ is a messaging Queue which is used by some components of the applications.

**Versions of the DB Software you will get context from the developer, Meaning we need to check with developer.**
**Developer has shared the version information as RabbitMQ-3.x**
Setup the RabbitMQ repo file. You can create file using
```
vim /etc/yum.repos.d/rabbitmq.repo
```
``` shell title=/etc/yum.repos.d/rabbitmq.repo
[modern-erlang]
name=modern-erlang-el9
baseurl=https://yum1.novemberain.com/erlang/el/9/$basearch
        https://yum2.novemberain.com/erlang/el/9/$basearch
        https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/rpm/el/9/$basearch
enabled=1
gpgcheck=0

[modern-erlang-noarch]
name=modern-erlang-el9-noarch
baseurl=https://yum1.novemberain.com/erlang/el/9/noarch
        https://yum2.novemberain.com/erlang/el/9/noarch
        https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/rpm/el/9/noarch
enabled=1
gpgcheck=0

[modern-erlang-source]
name=modern-erlang-el9-source
baseurl=https://yum1.novemberain.com/erlang/el/9/SRPMS
        https://yum2.novemberain.com/erlang/el/9/SRPMS
        https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/rpm/el/9/SRPMS
enabled=1
gpgcheck=0

[rabbitmq-el9]
name=rabbitmq-el9
baseurl=https://yum2.novemberain.com/rabbitmq/el/9/$basearch
        https://yum1.novemberain.com/rabbitmq/el/9/$basearch
        https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-server/rpm/el/9/$basearch
enabled=1
gpgcheck=0

[rabbitmq-el9-noarch]
name=rabbitmq-el9-noarch
baseurl=https://yum2.novemberain.com/rabbitmq/el/9/noarch
        https://yum1.novemberain.com/rabbitmq/el/9/noarch
        https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-server/rpm/el/9/noarch
enabled=1
gpgcheck=0
```

Install RabbitMQ 
```shell 
dnf install rabbitmq-server -y
```

Start RabbitMQ Service 
```shell 
systemctl enable rabbitmq-server
systemctl start rabbitmq-server
```

RabbitMQ comes with a default username / password as `guest/guest`. But this user cannot be used to connect. Hence, we need to create one user for the application.
```shell 
rabbitmqctl add_user roboshop roboshop123
rabbitmqctl set_permissions -p / roboshop ".*" ".*" ".*"
```
# 10-Payment 

This service is responsible for payments in RoboShop e-commerce app.
This service is written on `Python 3.x`, So need it to run this app.

Developer has chosen Python, Check with developer which version of Python is needed.
Install Python 3

```shell 
dnf install python3 gcc python3-devel -y
```

Configure the application.
Our application developed by the developer of our org and it is not having any RPM software just like other softwares. So we need to configure every step manually
We already discussed in Linux basics section that applications should run as nonroot user.
Add application User

```shell 
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
```

User **roboshop** is a function / daemon user to run the application. Apart from that we dont use this user to login to server.
Also, username **roboshop** has been picked because it more suits to our project name.
We keep application in one standard location. This is a usual practice that runs in the organization.
Lets setup an app directory.

```shell
mkdir /app 
```

Download the application code to created app directory.
```shell
curl -L -o /tmp/payment.zip https://roboshop-artifacts.s3.amazonaws.com/payment-v3.zip 
cd /app 
unzip /tmp/payment.zip
```

Every application is developed by development team will have some common softwares that they use as libraries. This application also have the same way of defined dependencies in the application configuration.
Lets download the dependencies.

```shell 
cd /app 
pip3 install -r requirements.txt
```

We need to setup a new service in **systemd** so `systemctl` can manage this service
We already discussed in linux basics that advantages of systemctl managing services, Hence we are taking that approach. Which is also a standard way in the OS.
Setup SystemD Payment Service

```unit file (systemd) title=/etc/systemd/system/payment.service
[Unit]
Description=Payment Service

[Service]
User=root
WorkingDirectory=/app
// highlight-start
Environment=CART_HOST=<CART-SERVER-IPADDRESS>
Environment=CART_PORT=8080
Environment=USER_HOST=<USER-SERVER-IPADDRESS>
Environment=USER_PORT=8080
Environment=AMQP_HOST=<RABBITMQ-SERVER-IPADDRESS>
// highlight-end
Environment=AMQP_USER=roboshop
Environment=AMQP_PASS=roboshop123

ExecStart=/usr/local/bin/uwsgi --ini payment.ini
ExecStop=/bin/kill -9 $MAINPID
SyslogIdentifier=payment

[Install]
WantedBy=multi-user.target
```

Hint! You can create file by using **`vim /etc/systemd/system/payment.service`**
Load the service.

```shell 
systemctl daemon-reload
```

This above command is because we added a new service, We are telling systemd to reload so it will detect new service.
enable and Start the payment service.

```shell 
systemctl enable payment 
systemctl start payment
```
# 11-Dispatch 

Dispatch is the service which dispatches the product after purchase. It is written in GoLang, So wanted to install GoLang.
Developer has chosen GoLang, Check with developer which version of GoLang is needed.
Install GoLang

```shell 
dnf install golang -y
```

Configure the application.
Our application developed by the developer of our org and it is not having any RPM software just like other softwares. So we need to configure every step manually
We already discussed in Linux basics section that applications should run as nonroot user.
Add application User

```shell 
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
```

User **roboshop** is a function / daemon user to run the application. Apart from that we dont use this user to login to server.
Also, username **roboshop** has been picked because it more suits to our project name.
We keep application in one standard location. This is a usual practice that runs in the organization.
Lets setup an app directory.

```shell
mkdir /app 
```

Download the application code to created app directory.
```shell
curl -L -o /tmp/dispatch.zip https://roboshop-artifacts.s3.amazonaws.com/dispatch-v3.zip 
cd /app 
unzip /tmp/dispatch.zip
```

Every application is developed by development team will have some common softwares that they use as libraries. This application also have the same way of defined dependencies in the application configuration.
Lets download the dependencies & build the software.

```shell 
cd /app 
go mod init dispatch
go get 
go build
```

We need to setup a new service in **systemd** so `systemctl` can manage this service
We already discussed in linux basics that advantages of systemctl managing services, Hence we are taking that approach. Which is also a standard way in the OS.
Setup SystemD Payment Service

```unit file (systemd) title=/etc/systemd/system/dispatch.service
[Unit]
Description = Dispatch Service
[Service]
User=roboshop
// highlight-start
Environment=AMQP_HOST=RABBITMQ-IP
// highlight-end
Environment=AMQP_USER=roboshop
Environment=AMQP_PASS=roboshop123
ExecStart=/app/dispatch
SyslogIdentifier=dispatch

[Install]
WantedBy=multi-user.target
```

Hint! You can create file by using **`vim /etc/systemd/system/dispatch.service`**
Load the service.

```shell 
systemctl daemon-reload
```

This above command is because we added a new service, We are telling systemd to reload so it will detect new service.
Enable and Start the dispatch service.

```shell 
systemctl enable dispatch 
systemctl start dispatch
```
