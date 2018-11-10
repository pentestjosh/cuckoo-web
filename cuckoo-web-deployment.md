# Cuckoo Web-Deployment with HTTPS and Basic-Authentication
A basic web deployment using uWSGI & nginx 

# Install pre-reqs
```
sudo apt-get install uwsgi uwsgi-plugin-python nginx
```

# uWSGI Setup
To begin, create a uWSGI configuration file at ``/etc/uwsgi/apps-available/cuckoo-web.ini`` that contains the actual configuration reported by `cuckoo web --uwsgi` command

It will look something like this:
```
cuckoo web --uwsgi
[uwsgi]
plugins = python
virtualenv = /home/cuckoo/cuckoo
module = cuckoo.web.web.wsgi
uid = cuckoo
gid = cuckoo
static-map = /static=/home/..somepath..
# If you're getting errors about the PYTHON_EGG_CACHE, then
# uncomment the following line and add some path that is
# writable from the defined user.
# env = PYTHON_EGG_CACHE=
env = CUCKOO_APP=web
env = CUCKOO_CWD=/home/..somepath..
```
Add the the output to the cuckoo-web.ini

# Enable the config and start the server
```
sudo ln -s /etc/uwsgi/apps-available/cuckoo-web.ini /etc/uwsgi/apps-enabled/
sudo service uwsgi start cuckoo-web    # or reload, if already running
```

# nginx setup and configuration
With the Web Interface server running in uWSGI, nginx can now be set up to run as a web server/reverse proxy, backending HTTP requests to it.

To begin, create a nginx configuration file at ``/etc/nginx/sites-available/cuckoo-web`` that contains the actual configuration as displayed by the `cuckoo web --nginx` command.

Should look similar to this:
```
cuckoo web --nginx
upstream _uwsgi_cuckoo_web {
    server unix:/run/uwsgi/app/cuckoo-web/socket;
}

server {
    listen localhost:8000;

    # Cuckoo Web Interface
    location / {
        client_max_body_size 1G;
        uwsgi_pass  _uwsgi_cuckoo_web;
        include     uwsgi_params;
    }
}
```
# Make sure that nginx can connect to the uWSGI socket by placing its user in the cuckoo group:
```
sudo adduser www-data cuckoo
```
# Enable the server config and start the server
```
sudo ln -s /etc/nginx/sites-available/cuckoo-web /etc/nginx/sites-enabled/
sudo service nginx start    # or reload, if already running
```

# At this point, the Web Interface server should be available at port 8000 on the server.
`From this point forward, start Cuckoo with the following two commands, or add to run at boot` 
```
sudo service uwsgi start cuckoo-web
sudo service nginx start 
```

# REMEMBER - NO HTTPS AND NO AUTHENTICATION BY DEFAULT - LET'S SET THAT UP NOW !
`For this implementation we will be using a self-signed certificate`

# Setting up SSL with a self-signed certificate for nginx
Create a self-signed certificate
We can create a self-signed key and certificate pair with OpenSSL in a single command:
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
```
You will be asked a series of questions. Fill out the prompts accordingly.
The most important line is the one that requests the Common Name (e.g. server FQDN or YOUR name). 
You need to enter the domain name associated with your server or, more likely, your server's public IP address.

Both of the files you created will be placed in the appropriate subdirectories of the `/etc/ssl` directory.

# While we are using OpenSSL, we should also create a strong Diffie-Hellman group
```
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```
This may take a few minutes, but when it's done you will have a strong DH group at `/etc/ssl/certs/dhparam.pem` that we can use in our configuration.

We have created our key and certificate files under the /etc/ssl directory. Now we just need to modify our Nginx configuration to take advantage of these.

# Configure nginx to use SSL

# Create a Configuration Snippet Pointing to the SSL Key and Certificate
First, let's create a new Nginx configuration snippet in the `/etc/nginx/snippets` directory.

To properly distinguish the purpose of this file, let's call it `self-signed.conf`:
```
sudo nano /etc/nginx/snippets/self-signed.conf
```
Within this file, we just need to set the `ssl_certificate` directive to our certificate file and the `ssl_certificate_key` 
to the associated key. In our case, this will look like this:
```
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```
Add those lines and save and close the file.

# Create a Configuration Snippet with Strong Encryption Settings
Next, we will create another snippet that will define some SSL settings. This will set 
Nginx up with a strong SSL cipher suite and enable some advanced features that will help keep our server secure.
```
sudo nano /etc/nginx/snippets/ssl-params.conf
```
We will insert the following into the `ssl-params.conf` (this configuration is secure but does not support a ciphersuite for older legacy clients)
First, we will add our preferred DNS resolver for upstream requests. We will use Google's for this guide. We will also go ahead and set the ssl_dhparam 
setting to point to the Diffie-Hellman file we generated earlier.
```
# from https://cipherli.st/
# and https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html

ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
ssl_ecdh_curve secp384r1;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
# Disable preloading HSTS for now.  You can use the commented out header line that includes
# the "preload" directive if you understand the implications.
#add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;

ssl_dhparam /etc/ssl/certs/dhparam.pem;
```
Because we are using a self-signed certificate, the SSL stapling will not be used. 
Nginx will simply output a warning, disable stapling for our self-signed cert, and continue to operate correctly.

Save and close the file when you are finished.

# Adjust the Nginx Configuration to Use SSL
Now that we have our snippets, we can adjust our Nginx configuration to enable SSL.
Before we go any further, let's back up our current server block file:
```
sudo cp /etc/nginx/sites-available/cuckoo-web /etc/nginx/sites-available/cuckoo-web.bak
```
Now open the server block to make adjustments 
```
sudo nano /etc/nginx/sites-available/cuckoo-web
```
Inside, your server block, change it to look like this:
```
upstream _uwsgi_cuckoo_web {
    server unix:/run/uwsgi/app/cuckoo-web/socket;
}

server {
    listen x.x.x.x:8000;
    server_name x.x.x.x;
    return 301 https://$server_name$request_uri;
}    
   
    server {

    # SSL configuration

    listen 443 ssl http2 x.x.x.x;
    listen [::]:443 ssl http2 x.x.x.x;
    include snippets/self-signed.conf;
    include snippets/ssl-params.conf;

 # Cuckoo Web Interface
    location / {
        client_max_body_size 1G;
        proxy_redirect off;
        proxy_set_header X-Forwarded-Proto $scheme;
        uwsgi_pass  _uwsgi_cuckoo_web;
        include     uwsgi_params;
        #auth_basic "Private Property";
        #auth_basic_user_file /etc/nginx/.htpasswd;
    }
}    
```
Save and close the file when you are finished.

# Enable the Changes in Nginx
```
sudo nginx -t
```
If everything is successful, you will get a result that looks like this:
```
nginx: [warn] "ssl_stapling" ignored, issuer certificate not found
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
If it looks good, restart nginx 
```
sudo systemctl restart nginx
```
Test the SSL by firing up a browser to https://x.x.x.x and enjoy your HTTPS :)

# Time to set up Basic-Authentication 
We will need the htpasswd program from `apache2-utils'
```
sudo apt-get install apache2-utils
```

# Setting Up HTTP Basic Authentication Credentials
```
sudo htpasswd -c /etc/nginx/.htpasswd [USERNAME]
```
You will be asked to enter a password. It will store this hashed pw in the .htpasswd file. 

# Update your nginx configuration

Add the two lines to the location block 
```
sudo nano /etc/nginx/sites-available/cuckoo-web
```
Add:
```
auth_basic "Private Property";
auth_basic_user_file /etc/nginx/.htpasswd;
```
Save and close the file. 

# Reload nginx to test the setup
```
sudo service nginx reload
```

# Congrats, you now have a web-deployment with SSL and Basic-Authentication ! 







