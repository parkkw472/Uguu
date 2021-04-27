# What is Uguu?

Uguu is a simple lightweight temporary file uploading and sharing platform where files get deleted after X amount of time.

## Features

- One click uploading, no registration required
- A minimal, modern web interface
- Drag & drop supported
- Upload API with multiple response choices
  - JSON
  - HTML
  - Text
  - CSV
- Supports [ShareX](https://getsharex.com/) and other screenshot tools

### Demo

See the real world example at [uguu.se](https://uguu.se).

## Requirements

Original development environment is Nginx + PHP5.3 + SQLite, but is confirmed to
work with Apache 2.4 and newer PHP versions like PHP7.3.

## Install

For the purposes of this guide, we won't cover setting up Nginx, PHP, SQLite,
Node, or NPM. So we'll just assume you already have them all running well.

**NPM/Node is only needed to compile the files, Uguu runs on PHP.**

### Compiling

First you must get a copy of the uguu code.  To do so, clone this git repo.
```bash
git clone https://github.com/nokonoko/uguu
```

Assuming you already have Node and NPM working, compilation is easy.

Run the following commands to do so, please configure `dist.json` before you compile.
```bash
cd uguu/
make
make install
```
OR
```bash
make install DESTDIR=/desired/path/for/site
```
After this, the uguu site is now compressed and set up inside `dist/`, or, if specified, `DESTDIR`.

## Configuring

Front-end related settings, such as the name of the site, and maximum allowable
file size, are found in `dist.json`.  Changes made here will
only take effect after rebuilding the site pages.  This may be done by running
`make` from the root of the site directory.

Back-end related settings, such as database configuration, and path for uploaded files, are found in `includes/settings.inc.php`.  Changes made here take effect immediately. Change the following settings:
```php
define('UGUU_DB_CONN', 'sqlite:/path/to/db/uguu.sq3');
define('UGUU_FILES_ROOT', '/path/to/file/');
define('UGUU_URL', 'https://subdomainforyourfiles.your.site');
```

If you intend to allow uploading files larger than 2 MB, you may also need to
increase POST size limits in `php.ini` and webserver configuration. For PHP,
modify `upload_max_filesize` and `post_max_size` values. The configuration
option for nginx webserver is `client_max_body_size`.

Edit checkdb.sh and checkfiles.sh to the proper paths:
```bash
sqlite3 /path/to/db/uguu.sq3 "DELETE FROM files WHERE date <= strftime('%s', datetime('now', '-1 day'));"
```
```bash
find /path/to/files/ -mmin +1440 -exec rm -f {} \;
```
Then add them to your crontab:
```bash
0,30 * * * * bash /path/to/checkfiles.sh
0,30 * * * * bash /path/to/checkdb.sh
```

These scripts check if DB entries and files are older then 24 hours and if they are deletes them.

## MIME/EXT Blocking

Blocking certain filetypes from being uploaded can be changed by editing the following settings in `includes/settings.inc.php`:
```php
define('CONFIG_BLOCKED_EXTENSIONS', serialize(['exe', 'scr', 'com', 'vbs', 'bat', 'cmd', 'htm', 'html', 'jar', 'msi', 'apk', 'phtml']));
define('CONFIG_BLOCKED_MIME', serialize(['application/msword', 'text/html', 'application/x-dosexec', 'application/java', 'application/java-archive', 'application/x-executable', 'application/x-mach-binary']));
```

By default the most common malicious filetypes are blocked.

## Using SQLite as DB engine

We need to create the SQLite database before it may be used by uguu.
Fortunately, this is incredibly simple.  

First create a directory for the database, e.g. `mkdir /var/db/uguu`.  
Then, create a new SQLite database from the schema, e.g. `sqlite3 /var/db/uguu/uguu.sq3 -init /home/uguu/sqlite_schema.sql`.
Then, finally, ensure the permissions are correct, e.g.
```bash
chown www-data:www-data /var/db/uguu
chmod 0750 /var/db/uguu
chmod 0640 /var/db/uguu/uguu.sq3
```

Finally, edit `includes/settings.inc.php` to indicate this is the database engine you would like to use.  Make the changes outlined below
```php
define('UGUU_DB_CONN', '[stuff]'); ---> define('UGUU_DB_CONN', 'sqlite:/var/db/uguu/uguu.sq3');
define('UGUU_DB_USER', '[stuff]'); ---> define('UGUU_DB_USER', null);
define('UGUU_DB_PASS', '[stuff]'); ---> define('UGUU_DB_PASS', null);
```

*NOTE: The directory where the SQLite database is stored, must be writable by the web server user*

## Nginx example config

I won't cover settings everything up, here are some Nginx examples. Use [Letsencrypt](https://letsencrypt.org) to obain a SSL cert.

Main domain:
```
server{
    
    listen	        443 ssl http2;
    server_name		www.yourdomain.com yourdomain.com;

    ssl on;
    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/toprivkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;   

    root /path/to/uguu/dist/;
    autoindex		off;
    access_log      off;
    index index.html index.php;  

    location ~* \.(ico|css|js|ttf)$ {
    expires 7d;
    }

    location ~* \.php$ {
    fastcgi_pass unix:/var/run/php/php7.3-fpm.sock;
    fastcgi_intercept_errors on;
    fastcgi_index index.php;
    fastcgi_split_path_info ^(.+\.php)(.*)$;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

Subdomain serving files (do not enable PHP here):
```
server{
    listen          443 ssl http2;
    server_name     www.subdomain.serveryourfiles.com subdomain.serveryourfiles.com;

    ssl on;
    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
        
    root            /path/where/uploaded/files/are/stored/;
    autoindex       off;
    access_log	    off;
    index           index.html;
}
```

To redirect HTTP to HTTPS make a config for each domain like so:
```
server {
    listen 80;
    server_name www.domain.com domain.com; 
    return 301 https://domain.com$request_uri;
}
```

## API
To upload using curl or make a tool you can post using: 
```
curl -i -F files[]=@yourfile.jpeg https://uguu.se/upload.php (JSON Response)
```
```
curl -i -F files[]=@yourfile.jpeg https://uguu.se/upload.php?output=text (Text Response)
```
```
curl -i -F files[]=@yourfile.jpeg https://uguu.se/upload.php?output=csv (CSV Response)
```
```
curl -i -F files[]=@yourfile.jpeg https://uguu.se/upload.php?output=html (HTML Response)
```

## Getting help

Hit me up at [@nekunekus](https://twitter.com/nekunekus) or email me at neku@pomf.se

## Credits

Uguu is based on [Pomf](http://github.com/pomf/pomf).

## License

Uguu is free software, and is released under the terms of the Expat license. See
`LICENSE`.
