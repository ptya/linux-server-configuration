# Linux Server Configuration Udacity Project
This project is about to take a baseline installation of a Linux server and prepare it to host web applications.
Securing the server from a number of attack vectors, installing and configuring a database server, and deploying one of my existing web application onto it.

## Server Details
- Server: Linux-based [Amazon Lightsail](https://aws.amazon.com/lightsail/) instance
- OS: Ubuntu 16.04.3 LTS
- Public IP: 35.156.112.237
- SSH Port: 2200
- URL: [http://ec2-35-156-112-237.eu-central-1.compute.amazonaws.com](http://ec2-35-156-112-237.eu-central-1.compute.amazonaws.com)
- Deployed app: [Item Catalog](https://github.com/ptya/Item_catalog_applicaton)

## Server Configuration
### Server Security
- All currently installed packages have been updated.
- SSH port changed from default 22 to 2200 and Lightsail firewall configured accordingly.
- Uncomplicated Firewall (UFW) configured to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

### User Management
- Created user `grader` with sudoer access.
- SSH key generated for user `grader`.

### Database Setup
- The application is using PostgreSQL.
- Remote connections are disabled.
- Database `catalog` created and restricted to database user `catalog`.

### Installed packages
- [Python 3.5.2](https://www.python.org)
- [Apache 2.4.18](https://httpd.apache.org/)
- [mod_wsgi 4.3.0-1](http://modwsgi.readthedocs.io/en/develop/)
- [PostgreSQL 9.5](https://www.postgresql.org/)
- [pip 9.0.1](https://pip.pypa.io/en/stable/)
- [Virtualenv 15.1.0](https://virtualenv.pypa.io/)
- [Flask 0.12.2](http://flask.pocoo.org/)

## Application Configuration
An Apache virtual host is setup to listen on port 80.
WSGIDaemonProcess runs with www-data user using a python virtual environment.

### Virtual Host Configuration
Virtual Host file name `item-catalog.conf`.
```
<VirtualHost *:80>
    ServerName  http://ec2-18-195-247-12.eu-central-1.compute.amazonaws.com
    ServerAdmin email@address
    DocumentRoot /var/www/item-catalog/catalog
    #Location of the items-catalog WSGI file
    WSGIDaemonProcess webtool user=www-data group=www-data threads=5 home=/var/www/item-catalog/catalog/
    WSGIScriptAlias / /var/www/item-catalog/catalog/run.wsgi
    #Allow Apache to serve the WSGI app from our catalog directory
    <Directory /var/www/item-catalog/catalog>
        WSGIProcessGroup webtool
        WSGIApplicationGroup %{GLOBAL}
        WSGIScriptReloading On
        Order allow,deny
        Allow from all
    </Directory>
    #Allow Apache to deploy static content
    <Directory /var/www/item-catalog/catalog/app/static>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel debug
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

### WSGI script
The Python application and WSGI file is stored in `/var/www/item-catalog/catalog/`.

```
import sys

sys.path.insert(0, '/var/www/item-catalog/catalog')
activate_this = '/var/www/item-catalog/catalog/virtualenv/bin/activate_this.py'
with open(activate_this) as afile:
    exec(afile.read(), dict(__file__=activate_this))

from app import app as application
```

### Updates required on the Item Catalog application
Due to being deployed on an actual web server the following edits have been needed:
#### Config file changes
`/var/www/item-catalog/catalog/instance/config.py`

##### Changing sqlite database to postgresql:
```
# SQLAlchemy
SQLALCHEMY_DATABASE_URI = 'postgresql://catalog:catalog@localhost/catalog'
```
##### Changing path to client secret files:
```
# OAUTH JSON files
GOO_CLIENT_ID = json.loads(
    open('%s/goo_client_secret.json' % BASEDIR, 'r')
    .read())['web']['client_id']

GH_CLIENT_ID = json.loads(
    open('%s/gh_client_secret.json' % BASEDIR, 'r')
    .read())['web']['client_id']

GH_SECRET = json.loads(
    open('%s/gh_client_secret.json' % BASEDIR, 'r')
    .read())['web']['client_secret']

FB_CLIENT_ID = json.loads(
    open('%s/fb_client_secret.json' % BASEDIR, 'r')
    .read())['web']['app_id']

FB_SECRET = json.loads(
    open('%s/fb_client_secret.json' % BASEDIR, 'r')
    .read())['web']['app_secret']
```
#####Changing path to upload folder:
```
UPLOAD_FOLDER = os.path.abspath('%s/../app/static/images/uploaded/' % BASEDIR)
```
#### File handling changes due to utilizing web server
`/var/www/item-catalog/catalog/app/items/views.py`
##### Delete item image check if None (line #177)
```
if deleted_item.image != '' and deleted_item.image is not None:
```
##### New item check if file is selected in form (lines #81-102)
```
if request.method == 'POST':
    filename = ''
    if 'file' in request.files:
        file = request.files['file']
        # clean filename for possible mischief
        filename = secure_filename(file.filename)
        if file and filename != '' and allowed_file(filename):
            # add hash folder for overwrite safety
            dirname = ''.join(random.choice(
                string.ascii_uppercase + string.digits) for x in range(8))
            path = os.path.join(app.config['UPLOAD_FOLDER'], dirname,
                                filename)
            if not os.path.exists(os.path.dirname(path)):
                try:
                    os.makedirs(os.path.dirname(path))
                # Guard against race condition
                except OSError as exc:
                    if exc.errno != errno.EEXIST:
                        raise
            file.save(path)
            # update filename to store in db
            filename = '%s/%s' % (dirname, filename)
```
##### Edit item check if file is selected in form (lines #125-154)
```
if request.method == 'POST':
    filename = ''
    if 'file' in request.files:
        file = request.files['file']
        # clean filename for possible mischief
        filename = secure_filename(file.filename)
        if file and filename != '' and allowed_file(filename):
            # add hash folder for overwrite safety
            dirname = ''.join(random.choice(
                string.ascii_uppercase + string.digits) for x in range(8))
            path = os.path.join(app.config['UPLOAD_FOLDER'], dirname,
                                filename)
            if not os.path.exists(os.path.dirname(path)):
                try:
                    os.makedirs(os.path.dirname(path))
                # Guard against race condition
                except OSError as exc:
                    if exc.errno != errno.EEXIST:
                        raise
            file.save(path)
            # update filename to store in db
            filename = '%s/%s' % (dirname, filename)
            # delete previous image if there was
            if edited_item.image != None and edited_item.image != '':
                os.remove(os.path.join(app.config['UPLOAD_FOLDER'],
                                    edited_item.image))
                prev_dirname = edited_item.image.split('/')[0]
                os.rmdir(os.path.join(app.config['UPLOAD_FOLDER'],
                                    prev_dirname))
```
#### Login related changes
- Updated OAuth providers redirect uri to the Lightsail instance's hostname.
- Removed `redirect_uri` parameter from github's redirec in `/var/www/item-catalog/catalog/app/static/js/login.js`

## Getting Started
### Connecting via SSH
```
ssh grader@35.156.112.237 -p 2200 -i ~/.ssh/grader-linux-project
```
### Accessing the web application
Open [this url](http://ec2-35-156-112-237.eu-central-1.compute.amazonaws.com). Google Chrome recommended for best experience.

## License
Copyright (c) 2018 Péter Szabó. All rights reserved.

This work is licensed under the terms of the MIT license.
For a copy, see <https://opensource.org/licenses/MIT>.