# Install nginx with pagespeed support on ubuntu 20.04 LTS   

Setup required build packages.
```
apt-get install dpkg-dev build-essential zlib1g-dev libpcre3 git libpcre3-dev unzip -y
```
Install nginx 
```
apt install nginx -y
```
Once the installation is completed, you can verify the installed version of Nginx with the following command:
```
nginx -v
``
You should see the Nginx version in the following output:
```
nginx version: nginx/1.18.0 (Ubuntu)
```
Once you are finished, you can proceed to the next step.
```
wget http://nginx.org/download/nginx-1.18.0.tar.gz
```
Once the download is completed, extract the downloaded file with the following command:
```
tar -xvzf nginx-1.18.0.tar.gz
```
Next, download the ngx_pagespeed source from the Git repository with the following command:
```
git clone https://github.com/apache/incubator-pagespeed-ngx.git
cd incubator-pagespeed-ngx
git checkout latest-stable
```
there will be a file now for psol library url 
```
cat PSOL_BINARY_URL
```
Download the libray
```
wget https://dl.google.com/dl/page-speed/psol/1.13.35.2-x64.tar.gz

tar -xvzf 1.13.35.2-x64.tar.gz
```
Next, change the directory to the Nginx source and install all required dependencies with the following command:
```
cd /root/nginx-1.18.0
apt-get build-dep nginx
```
Above will install dependent packages.

Next, compile the ngx_pagespeed module with the following command:
```
./configure --with-compat --add-dynamic-module=/root/incubator-pagespeed-ngx
```
Next, run the following command to build the Pagespeed module:
```
make modules
``
Now copy the compiled module
```
cp objs/ngx_pagespeed.so /usr/share/nginx/modules/
```

We can now modify our nginx config file to use ngx-pagespeed
```
vi /etc/nginx/nginx.conf
```
Add the following line at the beginning of the file:
```
load_module modules/ngx_pagespeed.so;
```
fix owenrship and setup for pagespeed cache
```
mkdir -p /var/ngx_pagespeed_cache

chown -R www-data:www-data /var/ngx_pagespeed_cache
```
Now open site vhost config and use it.
```
vi /etc/nginx/sites-available/default

server {
     listen 80;
     server_name example.com; 

     root /var/www/html;
     index index.nginx-debian.html index.html index.htm;

     access_log   /var/log/nginx/access.log;
     error_log    /var/log/nginx/error.log;

     location / {
         try_files $uri $uri/ =404;
     }

     pagespeed on;
     pagespeed FileCachePath "/var/ngx_pagespeed_cache/";
     pagespeed RewriteLevel OptimizeForBandwidth;

     location ~ ".pagespeed.([a-z].)?[a-z]{2}.[^.]{10}.[^.]+" {
         add_header "" "";
     }
```

All set and done . Enjoy!!
