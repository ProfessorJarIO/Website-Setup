# Setting Everything Up
> December 26th, 2024
------------------------------------------------------

1. Install yay

```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

2. Install openresty

```bash
yay -S openresty
```

Add the following line to the `~/.bashrc` or `~/.zshrc`:
```bash
export PATH=/opt/openresty/bin:$PATH

# Apply the changes via bashrc
source ~/.bashrc

# for zshrc
source ~/.zshrc
```

Now you should have `opm` and `openresty` added to your PATH

You need to start the service now:

```bash
sudo systemctl start openresty; sudo systemctl enable openresty
```

3. Install PHP, PHP-FPM, Composer, and PHP-pear

```bash
sudo pacman -S php php-fpm composer
yay -S php-pear
```

4. Configure `nginx.conf`

It is pivotal you set the `root` to the path of where you want your application to lie in. Make sure the `root`
derivative is not in individual `location`s, but rather in the `server` derivative.

Also add `index index.php` in the `server` derivative.

Add this for openresty to render and use PHP files:

```conf
location ~ \.php$ {
	    fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
	    fastcgi_index index.php;
	    include fastcgi.conf;
}
```

5. Install MongoDB


```bash
# mongodb-bin is just the server/DB that holds content. Mongodb from pecl is the PHP helper extension
yay -S mongodb-bin
sudo pecl install mongodb
```

Now we have to start/enable the service

```bash
sudo systemctl start mongodb
sudo systemctl enable mongodb
```

Now we have to create a user with admin privileges.

```bash
mongosh
```

In the mongo shell:

```bash
use admin

# then
db.createUser( { user: "[YOUR USERNAME]", pwd: "[YOUR PASSWORD]", roles: [{ role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase"] } )
```

Append the following to your `/etc/mongodb.conf`.
```txt
security:
  authorization: "enabled"
```

If you want, you can also allow remote connections. That way, you can connect from a different IP through Mongo Compass. Update the bindIp address to this:
```
bindIp: 0.0.0.0
```

Now restart mongodb:

```bash
sudo systemctl restart mongodb.service
```

From now on, you will use this command to log into your database:
```bash
mongo -u {username} -p {password} --authenticationDatabase admin
```
> https://www.linuxboost.com/how-to-install-and-configure-mongodb-on-arch-linux/

Create necessary directories for mongodb logs (not necessary on arch linux):
```bash
sudo mkdir -p /var/lib/mongodb
sudo mkdir -p /var/log/mongodb
sudo chown -R mongodb:mongodb /var/lib/mongodb
sudo chown -R mongodb:mongodb /var/log/mongodb
```

You will also need to ensure that the mongodb extension is enabled in your php.ini file. The location of your php.ini file will vary depending on your operating system. Add the following line to your php.ini file:

```bash
extension="mongodb.so"
```


6. Install Laravel

```bash
composer global require laravel/installer
```

7. Setup your project

```bash
composer create-project --prefer-dist laravel/laravel [LARAVEL PROJECT NAME]
```

Make sure you give the correct ownership:
```
sudo chown laravel-project http:muta
```

Fix this dumb permission error (I don't think this is necessary anymore since we changed the owner to http):
```
sudo chmod -R 775 /opt/openresty/nginx/html/[LARAVEL PROJECT NAME]/storage
```

Remember to change the nginx.conf file to make the root the `public` directory.

```
listen 80;
server_name localhost;

#charset koi8-r;

#access_log logs/host.access.log main;

root html/extrovert/public;
index index.php;

location / {
    #root html;
    #index index.html index.htm;
    try_files $uri $uri/ /index.php?$query_string;
}

location ~ /\.ht {
    deny all;
}

#error_page 404 /404.html;

# redirect server error pages to the static page /50x.html
#
#error_page 500 502 503 504 /50x.html;
#location = /50x.html {
#    root html;
#}

location ~ \.php$ {
    fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
    fastcgi_index index.php;
    #root /usr/share/nginx/html;
    include fastcgi.conf;
}

```

Make sure you install the necessary library for MongoDB to work with Laravel:
```bash
composer require mongodb/laravel-mongodb
```

8. Install Redis and Redis PHP Extension

```bash
sudo pacman -S redis php-redis
```

Then start/enable `redis`
```bash
sudo systemctl start redis
sudo systemctl enable redis
```

Now it's time we add authencation in redis service. First, edit `/etc/redis/redis.conf`:
```bash
sudo vim /etc/redis/redis.conf
```

Find the requirepass directive: Look for the line that says `# requirepass foobared` and uncomment it by removing the # at the beginning of the line

Set your desired password: Replace `foobared` with your desired password

Save changes, and restart redis service.

Authenticate using the new password: Open a new terminal and connect to Redis:

`redis-cli`

Then, authenticate using the `AUTH` command:

```bash
AUTH yourpasswordhere
```

Now you need to install the PHP extension helper for redis:

```bash
sudo pecl install redis
```

Enable the Redis extension: Add the following line to your php.ini file to enable the Redis extension:
```bash
extension=redis.so
```


9. Configure `.env` file to use Mongodb and Redis

Redis:
```
##
# READ THIS: IF YOU USE REDIS, YOU WILL NO LONGER SEE ANYTHING IN SESSIONS TABLE IN DB. THIS IS OKAY, BECAUSE WE ARE NOW USING REDIS, WHICH IS FASTER
# AT FIRST, I GOT CONFUSED ABOUT THIS, BUT NOW I REALIZED ALL SESSION DATA WAS BEING STORED IN REDIS
##
SESSION_DRIVER=redis
SESSION_LIFETIME=120
SESSION_ENCRYPT=false
SESSION_PATH=/
SESSION_DOMAIN=null


REDIS_CLIENT=phpredis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=password
REDIS_PORT=6379

```

Mongodb:

.env file:
```
DB_CONNECTION=mongodb
MONGODB_HOST=127.0.0.1
MONGODB_PORT=27017
MONGODB_DATABASE=myEX_laravel
MONGODB_USERNAME=muta
MONGODB_PASSWORD=password
```

in `config/database.php` and under `connections`:

```
'mongodb' => [
    'driver'   => 'mongodb',
    'dsn'      => env('MONGODB_URI', 'mongodb://'.env('MONGODB_HOST', '127.0.0.1').':'.env('MONGODB_PORT', 27017)),
    'database' => env('MONGODB_DATABASE', 'your_database_name'),
    'username' => env('MONGODB_USERNAME', 'your_username'),
    'password' => env('MONGODB_PASSWORD', 'your_password'),
    'options'  => [
        'database' => 'admin' // This is required if you're using MongoDB 3.0 or higher
    ]
],
```
