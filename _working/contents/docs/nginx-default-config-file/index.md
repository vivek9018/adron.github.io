---
title: Nginx Default Config File
author: Adron Hall
date: 2016-07-15:15:43:42
template: doc.jade
---
The following is a default Nginx configuration file as referred to in my [NGinx Notes from URL Redirect Project on Google Cloud with Terraform & Packer](../../articles/nginx-notes-from-the-url-redirect/) post.

	user       www www;  ## Default: nobody
	worker_processes  5;  ## Default: 1
	error_log  logs/error.log;
	pid        logs/nginx.pid;
	worker_rlimit_nofile 8192;

	events {
	  worker_connections  4096;  ## Default: 1024
	}

	http {
	  include    conf/mime.types;
	  include    /etc/nginx/proxy.conf;
	  include    /etc/nginx/fastcgi.conf;
	  index    index.html index.htm index.php;

	  default_type application/octet-stream;
	  log_format   main '$remote_addr - $remote_user [$time_local]  $status '
	    '"$request" $body_bytes_sent "$http_referer" '
	    '"$http_user_agent" "$http_x_forwarded_for"';
	  access_log   logs/access.log  main;
	  sendfile     on;
	  tcp_nopush   on;
	  server_names_hash_bucket_size 128; # this seems to be required for some vhosts

	  server { # php/fastcgi
	    listen       80;
	    server_name  domain1.com www.domain1.com;
	    access_log   logs/domain1.access.log  main;
	    root         html;

	    location ~ \.php$ {
	      fastcgi_pass   127.0.0.1:1025;
	    }
	  }

	  server { # simple reverse-proxy
	    listen       80;
	    server_name  domain2.com www.domain2.com;
	    access_log   logs/domain2.access.log  main;

	    # serve static files
	    location ~ ^/(images|javascript|js|css|flash|media|static)/  {
	      root    /var/www/virtual/big.server.com/htdocs;
	      expires 30d;
	    }

	    # pass requests for dynamic content to rails/turbogears/zope, et al
	    location / {
	      proxy_pass      http://127.0.0.1:8080;
	    }
	  }

	  upstream big_server_com {
	    server 127.0.0.3:8000 weight=5;
	    server 127.0.0.3:8001 weight=5;
	    server 192.168.0.1:8000;
	    server 192.168.0.1:8001;
	  }

	  server { # simple load balancing
	    listen          80;
	    server_name     big.server.com;
	    access_log      logs/big.server.access.log main;

	    location / {
	      proxy_pass      http://big_server_com;
	    }
	  }
	}

Another sample file is shown below.

	#user  nobody;
	#Defines which Linux system user will own and run the Nginx server

	worker_processes  1;
	#Referes to single threaded process. Generally set to be equal to the number of CPUs or cores.

	#error_log  logs/error.log; #error_log  logs/error.log  notice;
	#Specifies the file where server logs. 

	#pid        logs/nginx.pid;
	#nginx will write its master process ID(PID).

	events {
	    worker_connections  1024;
	    # worker_processes and worker_connections allows you to calculate maxclients value: 
	    # max_clients = worker_processes * worker_connections
	}


	http {
	    include       mime.types;
	    # anything written in /opt/nginx/conf/mime.types is interpreted as if written inside the http { } block

	    default_type  application/octet-stream;
	    #

	    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
	    #                  '$status $body_bytes_sent "$http_referer" '
	    #                  '"$http_user_agent" "$http_x_forwarded_for"';

	    #access_log  logs/access.log  main;

	    sendfile        on;
	    # If serving locally stored static files, sendfile is essential to speed up the server,
	    # But if using as reverse proxy one can deactivate it
	    
	    #tcp_nopush     on;
	    # works opposite to tcp_nodelay. Instead of optimizing delays, it optimizes the amount of data sent at once.

	    #keepalive_timeout  0;
	    keepalive_timeout  65;
	    # timeout during which a keep-alive client connection will stay open.

	    #gzip  on;
	    # tells the server to use on-the-fly gzip compression.

	    server {
	        # You would want to make a separate file with its own server block for each virtual domain
	        # on your server and then include them.
	        listen       80;
	        #tells Nginx the hostname and the TCP port where it should listen for HTTP connections.
	        # listen 80; is equivalent to listen *:80;
	        
	        server_name  localhost;
	        # lets you doname-based virtual hosting

	        #charset koi8-r;

	        #access_log  logs/host.access.log  main;

	        location / {
	            #The location setting lets you configure how nginx responds to requests for resources within the server.
	            root   html;
	            index  index.html index.htm;
	        }

	        #error_page  404              /404.html;

	        # redirect server error pages to the static page /50x.html
	        #
	        error_page   500 502 503 504  /50x.html;
	        location = /50x.html {
	            root   html;
	        }

	        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
	        #
	        #location ~ \.php$ {
	        #    proxy_pass   http://127.0.0.1;
	        #}

	        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
	        #
	        #location ~ \.php$ {
	        #    root           html;
	        #    fastcgi_pass   127.0.0.1:9000;
	        #    fastcgi_index  index.php;
	        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
	        #    include        fastcgi_params;
	        #}

	        # deny access to .htaccess files, if Apache's document root
	        # concurs with nginx's one
	        #
	        #location ~ /\.ht {
	        #    deny  all;
	        #}
	    }


	    # another virtual host using mix of IP-, name-, and port-based configuration
	    #
	    #server {
	    #    listen       8000;
	    #    listen       somename:8080;
	    #    server_name  somename  alias  another.alias;

	    #    location / {
	    #        root   html;
	    #        index  index.html index.htm;
	    #    }
	    #}


	    # HTTPS server
	    #
	    #server {
	    #    listen       443 ssl;
	    #    server_name  localhost;

	    #    ssl_certificate      cert.pem;
	    #    ssl_certificate_key  cert.key;

	    #    ssl_session_cache    shared:SSL:1m;
	    #    ssl_session_timeout  5m;

	    #    ssl_ciphers  HIGH:!aNULL:!MD5;
	    #    ssl_prefer_server_ciphers  on;

	    #    location / {
	    #        root   html;
	    #        index  index.html index.htm;
	    #    }
	    #}

	}