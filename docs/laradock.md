在寫 Laravel 之前的環境設置非常繁瑣，而且最近也在找能夠輕鬆架好多個 laravel 專案的方法，所以就來試試 Laradock，由於有些雷所以還是記錄一下

```
git clone https://github.com/Laradock/laradock.git
```

#### 修改 Laradock 下的 .env 
```
cp env-example .env
```
```
你專案的路徑
APP_CODE_PATH_HOST=/path/to/project
APP_CODE_PATH_CONTAINER=/var/www
```

如此 /path/to/project 會對應至 workspace 的 /var/www ，如果你一次要跑多個專案可以將他們都放到該資料夾下

#### Nginx

.env
```
NGINX_HOST_HTTP_PORT=80
NGINX_HOST_HTTPS_PORT=443
NGINX_SSL_PATH=./nginx/ssl/
```
nginx/sites 下編輯設定檔
```
$ cp laravel.conf.example laravel.test.conf
```
#### Mysql

.env: `MYSQL_USER`, `MYSQL_PASSWORD` 要和 laravel 下的 .env 一樣 
```
MYSQL_DATABASE=dataset
MYSQL_USER=default
MYSQL_PASSWORD=secret
MYSQL_ROOT_PASSWORD=
```

laravel .env: DB_HOST 改為 mysql
```
DB_HOST=mysql
```
 
如果你希望 root 密碼為空，在 docker-compose.yml 中新增 `MYSQL_ALLOW_EMPTY_PASSWORD=true`
```
environment:
- MYSQL_DATABASE=${MYSQL_DATABASE}
- MYSQL_USER=${MYSQL_USER}
- MYSQL_PASSWORD=${MYSQL_PASSWORD}
- MYSQL_ALLOW_EMPTY_PASSWORD=true
- MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
- TZ=${WORKSPACE_TIMEZONE}
```
#### phpmyadmin

.env: `PMA_USER`, `PMA_PASSWORD` 要和 laravel 下的 .env 一樣
```
PMA_USER=default
PMA_PASSWORD=secret
PMA_ROOT_PASSWORD=
...
PMA_PORT=8081
```

#### 啟動

第一次跑可能會有點久
```
docker-compose up -d nginx mysql phpmyadmin
```
* 錯誤: 需要將 docker 更新到最新版

    > Service 'mysql' failed to build: Please provide a source image with `from` prior to commit

#### 設定 laravel

等到環境都弄好後就進入 container 設定好 laravel project
```
$ docker-compose exec workspace bash
$ composer install
...
```
* migrate 錯誤1

    > php_network_getaddresses: getaddrinfo failed: nodename nor servname provided, or not known
    
    原因是本地不知道 mysql 是什麼，需要指定 DB_HOST 為 127.0.0.1

```
$ env DB_HOST=127.0.0.1 php artisan migrate
```

* migrate 錯誤2 

    > SQLSTATE[42000]: Syntax error or access violation: 1231 Variable 'sql_mode' can't be set to the value of 'NO_AUTO_CREATE_USER'
        
    需要修改 laravel project 的 config/database.php，將 mysql 的 stict 改為 false

### 跑多個專案

要將多個 project 放到同一台機器上也很簡單，首先在你的 APP_CODE_PATH_HOST 中放好多個 project
```
/path/to/project
- project1
- project2
```

然後去 Laradock/nginx/sites 下多建立多個設定擋

```
# project1.conf
server {
    listen 80;
    listen [::]:80;

    server_name project1;
    root /var/www/project1/public;
    ...
}
```

```
# project2.conf
server {
    listen 80;
    listen [::]:80;

    server_name project2;
    root /var/www/project2/public;
    ...
}
```

如果是在本地測試在 /etc/hosts 加上

```
127.0.0.1 project1
127.0.0.1 project2
```

再來要新增多個 database ，上網查了下好像沒有輕鬆的方法可以直接建立多個 database，需要在 Laradock/mysql/docker-entrypoint-initdb.d 下寫腳本讓 container 建立時執行，既然都要在這邊寫腳本了，乾脆所有會用到的資料庫都在這邊設定好，上述的設定擋就填個 dummy database 和 user 

```
# createdb.sql

CREATE DATABASE IF NOT EXISTS `project1`;
CREATE DATABASE IF NOT EXISTS `project2`;

CREATE USER 'user1'@'%' IDENTIFIED BY 'user1';
CREATE USER 'user2'@'%' IDENTIFIED BY 'user2';

GRANT ALL ON `project1`.* TO 'user1'@'%';
GRANT ALL ON `project2`.* TO 'user2'@'%';
```

記得在 mysql 設定有更動後刪掉資料再重建

```
$ rm -Rf ~/.laradock/data/mysql
```

---

如此就架好多個 laravel 專案拉，如果 https 需要憑證可以使用 cerbot，會自動幫你去 letsencrypt 申請憑證，再將 cerbot 存放憑證的位址和 nginx 連通即可。 
