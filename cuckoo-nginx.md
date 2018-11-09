# Cuckoo Web-Deployment
The default web server from Cuckoo provides no encryption (http only) and no authentication.
Let's fix that ! First we will set up uWSGI and nginx to serve and backend our http requests.

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
# REMEMBER - NO HTTPS AND NO AUTHENTICATION BY DEFAULT - YOU SHOULD SET THAT UP NOW !
