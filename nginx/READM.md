ps -C nginx -f
nginx -s reload 
netstat -tlnp | grep nginx  // ss -tlnp | grep 443
ngxtop
getenforce
setenforce 0 
semanage fcontext -l | grep /user/share/nginx/html
semanage fcontext -a -t httpd_sys_content_t "/var/www(/.*)?"
proxy_set_header X-Real-IP $remote_addr;

in http block
proxy_cache_path /var/cache/nginx
levels=1:2
keys_zone=my_cache:10m
max_size=100m
inactive=60m;

steps : **( nginx -t )** >> **( /etc/nginx/nginx.conf )**

**add server 1**
1- mkdir -p /var/www/example.com
2- vim /var/www/example.com/index.html >> welcome example.com
3- vim /var/www/example.com/404.html >> error page 
4- chown -R nginx:nginx /var/www/example.com
5- vim /etc/nginx/conf.d/example.com.config >>
     server {
        listen       80;
        listen       [::]:80;
        server_name   example.com;
        root         /var/www/example.com;
        index        index.html
        error_page 404 /404.html;
        location = /404.html {
         root     /var/www/example.com
         index     404.html
        }
         #redirect /oldpage to /newpage
            location /oldpage {
            rewrite ^/oldpage$ /newpage permanent;
         }
    }


**add server 2**
1- mkdir -p /var/www/test.com/files
2- vim /var/www/test.com/index.com >> welcome index 
3- chown -R nginx:nginx /var/www/test.com
4- vim /etc/nginx/conf.d/test.com.conf  >>
  server {
        listen       80;
        listen       [::]:80;
        server_name   test.com;
        root         /var/www/test.com;
        index        index.html

      #enable dir listing for /files/
       location /files/{
            autoindex on;
          }
     
      }
5- vim /etc/nginx/conf.d/oldsite.com.conf
    server {
      listen 80;
      server_name oldsite.com;
      return 301 http://exapmle.com$request_uri;
    }
6- vim /etc/hosts 
    192.168.105.50 example.com test.com oldsite.com

**ssl**
1- mkdir -p /etc/nginx/ssl
2- cd /etc/nginx/ssl
3- openssl genrsa -out example.key 2048
4-openssl req -new -x509 -key example.key  -out example.crt   -days 365
5- vim /etc/nginx/conf.d/exapmle.com.conf >
server {
        listen 80;
        server_name   example.com;
        return 301 https://$host$request_uri;
}
server{
        listen 443 ssl;
        server_name   example.com;
        root         /var/www/example.com;
        index        index.html;
        add_header Strict-Transport-Security "max-age=3153000; includSubDomains" always;

        ssl_certificate  /etc/nginx/ssl/example.crt;
        ssl_certificate_key /etc/nginx/ssl/example.key;
    }


**proxy and loadbalancer**
upstream backend.server {
      server backend1.example.com;
      server backend2.example.com;
      server backend3.example.com;
}
server {
      listen 80;
      server_name   example.com;
      location /{
            proxy_pass http://backend.server;
            proxy_connect_timeout 60s;
            proxy_read_timeout  120s;
      }
      error_page 502 503 = @fallback;
}
location @fallback{
      return 200 "the server unavailable "
}


/////
upstream backend.server {
server backend2.example.com;
server backend1.example.com;
}

server {
        listen 80;
        server_name   example.com;

        location / {
    proxy_cache my_cache;
    proxy_cache_valid 200 10m;
          proxy_pass http://backend.server;
           proxy_http_version 1.1;
           proxy_set_header Connection "";
        }
}
server {
        listen 443 ssl;
        server_name   example.com;
        location / {
       proxy_cache my_cache;
         proxy_cache_valid 200 10m;
          proxy_pass http://backend.server;
          proxy_http_version 1.1;
          proxy_set_header Connection "";
        }
        add_header Strict-Transport-Security "max-age=3153000; includSubDomains" always;

        ssl_certificate  /etc/nginx/ssl/example.crt;
        ssl_certificate_key /etc/nginx/ssl/example.key;
    }