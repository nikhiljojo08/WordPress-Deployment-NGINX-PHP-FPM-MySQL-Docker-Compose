# WordPress-Deployment-NGINX-PHP-FPM-MySQL-Docker-Compose
WordPress Deployment with NGINX, PHP-FPM and MySQL using Docker Compose

# 1. Add MySQL configuration

Create a docker-compose.yml file and add the below configuration to the file. The configuration for MySQL database is listed under the service name database.


    version: '3'

      services:

        database:
        
            image: mysql:5.6
        
            container_name: database

            volumes:
               - mysql-data:/var/lib/mysql/
        
            networks:
               - wpnet

            environment:
               - MYSQL_ROOT_PASSWORD=root@123
               - MYSQL_DATABASE=wordpress
               - MYSQL_USER=wordpress
               - MYSQL_PASSWORD=wordpress

            restart: always

The below section mounts the host machine path mysql-data in the container path /var/lib/mysql.
           
           volumes:
               - mysql-data:/var/lib/mysql/

# 2. Add WordPress Configuration

Add WordPress configuration to the docker-compose.yml file as a service named wordpress.


    version: '3'

      services:

        database:
        
            image: mysql:5.6
        
            container_name: database

            volumes:
               - mysql-data:/var/lib/mysql/
        
            networks:
               - wpnet

            environment:
               - MYSQL_ROOT_PASSWORD=root@123
               - MYSQL_DATABASE=wordpress
               - MYSQL_USER=wordpress
               - MYSQL_PASSWORD=wordpress

            restart: always
		   
        wordpress:
        
		    image: wordpress:php7.3-fpm-alpine
        
		    container_name: wordpress
            
			volumes:
                - wp-data:/var/www/html/

            networks:
                - wpnet

            environment:

               - WORDPRESS_DB_HOST=database
               - WORDPRESS_DB_USER=wordpress
               - WORDPRESS_DB_PASSWORD=wordpress
               - WORDPRESS_DB_NAME=wordpress
            restart: always
    

Make sure the WordPress environment variables MYSQL_ROOT_PASSWORD, WORDPRESS_DB_NAME, WORDPRESS_DB_USER, and WORDPRESS_DB_PASSWORD match exactly with the MariaDB environment variables MYSQL_ROOT_PASSWORD, MYSQL_DATABASE, MYSQL_USER, and MYSQL_PASSWORD respectively. In case of any mismatch, the WordPress application would not be able to connect to the database.   

The below section mounts the host machine path wp-data in the container path /var/www/html.

            volumes:
                - wp-data:/var/www/html/
                
This enables all our WordPress application files to be stored in the host machine instead of the container.

# 3. Add NGINX Configuration

Add NGINX configuration to the docker-composer.yml file as a service named nginx.

    version: '3'

      services:

        database:
        
            image: mysql:5.6
        
            container_name: database

            volumes:
               - mysql-data:/var/lib/mysql/
        
            networks:
               - wpnet

            environment:
               - MYSQL_ROOT_PASSWORD=root@123
               - MYSQL_DATABASE=wordpress
               - MYSQL_USER=wordpress
               - MYSQL_PASSWORD=wordpress

            restart: always
		   
        wordpress:
        
		    image: wordpress:php7.3-fpm-alpine
        
		    container_name: wordpress
            
			volumes:
                - wp-data:/var/www/html/

            networks:
                - wpnet

            environment:

               - WORDPRESS_DB_HOST=database
               - WORDPRESS_DB_USER=wordpress
               - WORDPRESS_DB_PASSWORD=wordpress
               - WORDPRESS_DB_NAME=wordpress
            restart: always
	    
	    nginx:
            
            image: nginx:alpine
         
            container_name: nginx
            
	        volumes:
               - ./nginx:/etc/nginx/conf.d/
               - ./ssl:/etc/ssl/    
               - wp-data:/var/www/html/
        
		    networks:
               - wpnet
            ports:
               - 80:80
               - 443:443
            restart: always             
    
 The below section maps the host machine port 80 to the container port 80. Also, maps the host machine port 443 to the container port 443.
 
             ports:
               - 80:80
               - 443:443
               
We also need to provide a configuration file for the Nginx server. Create a directory named nginx and put the below contents in a file named nginx.conf in that directory.


     # default.conf
     # redirect to HTTPS
     server {
         listen 80;
         listen [::]:80;
         server_name $host;
        location / {
             # update port as needed for host mapped https
             rewrite ^ https://$host:443$request_uri? permanent;
         }
     }

     server {
         listen 443 ssl http2;
         listen [::]:443 ssl http2;
         server_name $host;
         index index.php index.html index.htm;
         root /var/www/html;
         server_tokens off;
         client_max_body_size 75M;

         # update ssl files as required by your deployment
         ssl_certificate     /etc/ssl/fullchain.pem;
         ssl_certificate_key /etc/ssl/privkey.pem;

         # logging
         access_log /var/log/nginx/wordpress.access.log;
         error_log  /var/log/nginx/wordpress.error.log;

         # some security headers ( optional )
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        add_header Content-Security-Policy "default-src * data: 'unsafe-eval' 'unsafe-inline'" always;

         location / {
             try_files $uri $uri/ /index.php$is_args$args;
         }

          location ~ \.php$ {
             try_files $uri = 404;
             fastcgi_split_path_info ^(.+\.php)(/.+)$;
             fastcgi_pass wordpress:9000;
             fastcgi_index index.php;
             include fastcgi_params;
             fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
             fastcgi_param PATH_INFO $fastcgi_path_info;
         }

         location ~ /\.ht {
             deny all;
         }

         location = /favicon.ico {
             log_not_found off; access_log off;
         }

         location = /favicon.svg {
             log_not_found off; access_log off;
         }

         location = /robots.txt {
             log_not_found off; access_log off; allow all;
         }

         location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
             expires max;
             log_not_found off;
         }
     }
     
     
  # 4. Install SSL Certificate.
  
  SSL  certificates are included in the repository ssl/ for demonstration purposes and you can replace it with genuine certificates.
  
     Certificate generation
     
     openssl req -x509 -outform pem -out chain.pem -keyout privkey.pem \
     -newkey rsa:4096 -nodes -sha256 -days 3650 \
     -subj '/CN=localhost' -extensions EXT -config <( \
      printf "[dn]\nCN=localhost\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:localhost\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")


  # 5. Deployment
  
The complete docker-compose.yml file given below.

   version: '3'

      services:

        database:
        
            image: mysql:5.6
        
            container_name: database

            volumes:
               - mysql-data:/var/lib/mysql/
        
            networks:
               - wpnet

            environment:
               - MYSQL_ROOT_PASSWORD=root@123
               - MYSQL_DATABASE=wordpress
               - MYSQL_USER=wordpress
               - MYSQL_PASSWORD=wordpress

            restart: always
		   
        wordpress:
        
		    image: wordpress:php7.3-fpm-alpine
        
		    container_name: wordpress
            
			volumes:
                - wp-data:/var/www/html/

            networks:
                - wpnet

            environment:

               - WORDPRESS_DB_HOST=database
               - WORDPRESS_DB_USER=wordpress
               - WORDPRESS_DB_PASSWORD=wordpress
               - WORDPRESS_DB_NAME=wordpress
            restart: always
	    
	    nginx:
            
            image: nginx:alpine
         
            container_name: nginx
            
	        volumes:
               - ./nginx:/etc/nginx/conf.d/
               - ./ssl:/etc/ssl/    
               - wp-data:/var/www/html/
        
		    networks:
               - wpnet
            ports:
               - 80:80
               - 443:443
            restart: always             
    
     networks:
        wpnet:
     
     volumes:
       mysql-data:
       wp-data:
       nginx:
       ssl:

Now create the containers and run the services with the below command.
    
    $ sudo docker-compose up -d
    
 Once the installation is over, open up the URL http://localhost in your browser. This should show the WordPress initial setup page.   
  
