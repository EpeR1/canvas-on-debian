# Canvas LMS

This is an unofficial install guide for [Canvas LMS](https://github.com/instructure/canvas-lms), on [Debian 10](https://www.debian.org/News/2019/20190706)  

## Hardware

Canvas requires ~8.2GB of RAM during building, and the Linux kernel uses ~2GB RAM.  
Canvas's modules are consuming at least 4GB of HDD.  
The absolute minimum hardware requirements is: **10GB RAM** and **6GB** free space on HDD! (Really!)
(And a Multi-Core CPU is beneficial)

## Canvas User

Set up or choose a user you want the Canvas Rails application to run as. This can be the same user as your webserver (www-data on Debian/Ubuntu), your personal user account, or something else. Once you've chosen or created a new user, you need to change the ownership of key files in your application root to that user.

    
    root@server:/$ useradd -p "*"  -m --home "/var/canvas" --shell "/bin/bash" --system "www-canvas" 
    root@server:/$ chage --mindays 0 --maxdays 99999 --warndays 7 "www-canvas"
    root@server:/$ chfn --full-name "www-canvas" "www-canvas"


## Dependency Installation (Debian 10+)

Canvas now requires Ruby, Postgresql, Apache2, etc...  

    root@server:/$ apt update
    root@server:/$ apt install ruby ruby-dev zlib1g-dev libxml2-dev \
            libsqlite3-dev postgresql libpq-dev \
            libxmlsec1-dev curl make g++ python \
            git gnupg sudo

Nodejs 10.x is required, and also available in Debian 10+  

    root@server:/$ apt install nodejs npm


## Yarn Installation

Canvas now prefers yarn instead of npm.  

* **Note #1:** As of 15Sep2018, the required yarn version is 1.10.1, specifically `sudo apt-get install yarn=1.10.1-1` )  
* **Note #2:** Since Yarn v1.22.5, `yarnpkg` is part of Yarn, but requires nodejs10+, which is available in Debian10+, so you don't need to install nodejs from external source!  

      root@server:/$ curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
      root@server:/$ echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
      root@server:/$ sudo apt update && sudo apt install yarn
 
 (If Canvas's folder is available now you can install the node modules also:)  

    root@server:/var/canvas$ yarn install
    
More info: [Yarn Classic installation](https://classic.yarnpkg.com/en/docs/install/#debian-stable)  
    
    

## Configuring Postgres

After installing Postgres, you will need to set your system username as a postgres superuser. You can do so by running the following commands:  

    root@server:/$ sudo -u postgres createuser $USER
    root@server:/$ sudo -u postgres psql -c "alter user $USER with superuser" postgres


You'll want to set up a Canvas user inside of Postgres. Note that in the below commands, you'll want to replace localhost with the hostname of the server Canvas is running on, if Canvas is running on a different server than Postgres.  

    # createuser will prompt you for a password for database user
    root@server:/$ sudo -u postgres createuser canvas --no-createdb \
                          --no-superuser --no-createrole --pwprompt
    root@server:/$ sudo -u postgres createdb canvas_production --owner=canvas

    
    
    
## Getting the code

The Canvas's code location, where it will run from, in this example will be: `/var/canvas`, but you can choose something else.  
We'll be referring to `/var/canvas` (or whatever you chose) as your Rails application root.  

### Using Git

Once you have a copy of Git installed on your system, getting the latest source for Canvas is as simple as checking out code from the repo, like so:

    root@server:/$ cd /var
    root@server:/$ git clone https://github.com/instructure/canvas-lms.git canvas
    root@server:/$ cd canvas
    root@server:/$ git checkout stable
    
    
## Dependency Installation  

### Ruby Gems
Most of Canvas' dependencies are Ruby Gems. Ruby Gems are a Ruby-specific package management system that operates orthogonally to operating-system package management systems.

### Bundler and Canvas dependencies

Canvas uses Bundler as an additional layer on top of Ruby Gems to manage versioned dependencies. Bundler is great!

    root@server:/var/canvas$  sudo gem install bundler --version 1.13.6
    root@server:/var/canvas$  bundle _1.13.6_ install --path vendor/bundle

    
### Yarn  

Then install the node modules:  

    root@server:/var/canvas$ yarn install


    
    
    
## Canvas default configuration

Before we set up all the tables in your database, our Rails code depends on a small few configuration files, which ship with good example settings, so, we'll want to set those up quickly. We'll be examining them more shortly. From the root of your Canvas tree, you can pull in the default configuration values like so:

    root@server:/var/canvas$  for config in amazon_s3 database delayed_jobs domain   \
                                            file_store outgoing_mail security external_migration;\
                                            do cp config/$config.yml.example config/$config.yml; done

### Dynamic settings configuration

This config file is useful if you don't want to run a consul cluster with canvas. Just provide the config data you would like for the DynamicSettings class to find, and it will use it whenever a call for consul data is issued. Data should be shaped like the example below, one key for the related set of data, and a hash of key/value pairs (no nesting)

    root@server:/var/canvas$  cp config/dynamic_settings.yml.example config/dynamic_settings.yml
    root@server:/var/canvas$  nano config/dynamic_settings.yml

### Database configuration

Now we need to set up your database configuration to point to your Postgres server and your production databases. Open the file *config/database.yml*, and find the **production** environment section. You can open this file with an editor like this:

    root@server:/var/canvas$  cp config/database.yml.example config/database.yml
    root@server:/var/canvas$  nano config/database.yml

Update this section to reflect your Postgres server's location and authentication credentials. This is the place you will put the password and database name, along with anything else you set up, from the Postgres setup steps.

### Outgoing mail configuration

For Canvas to work properly, you need an outgoing SMTP mail server. All you need to do is get valid outgoing SMTP settings. Open config/outgoing_mail.yml:

    root@server:/var/canvas$  cp config/outgoing_mail.yml.example config/outgoing_mail.yml
    root@server:/var/canvas$  nano config/outgoing_mail.yml

Find the **production** section and configure it to match your SMTP provider's settings. Note that the *domain* and *outgoing_address* fields are not for SMTP, but are for Canvas. *domain* is required, and is the domain name that outgoing emails are expected to come from. *outgoing_address* is optional, and if provided, will show up as the address in the *From* field of emails Canvas sends.

If you don't want to use authentication, simply comment out the lines for *user_name*, *password*, and *authentication*.


### URL configuration

In many notification emails, and other events that aren't triggered by a web request, Canvas needs to know the URL that it is visible from. For now, these are all constructed based off a domain name. Please edit the **production** section of *config/domain.yml* to be the appropriate domain name for your Canvas installation. For the *domain* field, this will be the part between `http://` and the next `/`. Instructure uses canvas.instructure.com.

    root@server:/var/canvas$  cp config/domain.yml.example config/domain.yml
    root@server:/var/canvas$  nano config/domain.yml

Note that the optional files_domain field is required if you plan to host user-uploaded files and wish to be secure, or if you want to allow custom Javascript in custom themes. files_domain must be a different hostname from the browser's perspective, even though it can be the same Apache server, and even the same IP address.

### Security configuration

You must insert randomized strings of at least 20 characters in this file:

    root@server:/var/canvas$  cp config/security.yml.example config/security.yml
    root@server:/var/canvas$  nano config/security.yml

    
    
    
## Generate Assets

Canvas needs to build a number of assets before it will work correctly. First, create the directories that will store the generated files. See the Canvas ownership section below in case you want to plan to assign ownership to canvasuser and the user does not exist yet.

    root@server:/$  cd /var/canvas
    root@server:/var/canvas$  mkdir -p log tmp/pids public/assets app/stylesheets/brandable_css_brands
    root@server:/var/canvas$  touch app/stylesheets/_brandable_variables_defaults_autogenerated.scss
    root@server:/var/canvas$  touch Gemfile.lock
    root@server:/var/canvas$  touch log/production.log
    root@server:/var/canvas$  chown -R root:root /var/canvas
    root@server:/var/canvas$  chown -R www-canvas config/environment.rb log tmp public/assets \
                              app/stylesheets/_brandable_variables_defaults_autogenerated.scss \
                              app/stylesheets/brandable_css_brands Gemfile.lock config.ru
Then will need to run:

    root@server:/var/canvas$  yarn install
    root@server:/var/canvas$  RAILS_ENV=production bundle exec rake canvas:compile_assets
    root@server:/var/canvas$  sudo chown -R www-canvas public/dist/brandable_css

 **Subsequent Updates** (NOT NEEDED FOR INITIAL DEPLOY)

If you are updating code, and you run canvas:compile_assets in a place that does not have a database connection, then you'll also want to run the following once code is in place and an active DB connection exists, in order to full update existing themes:  

    root@server:/var/canvas$  RAILS_ENV=production bundle exec rake brand_configs:generate_and_upload_all

    
    
## Database population

Once your database is configured, and assets are installed, we need to actually fill the database with tables and initial data. You can do this by running our rake migration and initialization tasks from your application's root:

    root@server:/var/canvas$  RAILS_ENV=production bundle exec rake db:initial_setup

Note that this initial setup will interactively prompt you to create an administrator account, the name for the default account, and whether to submit usage data to Instructure. The prompts can be "pre-filled" by setting the following environment variables:


|    Environment Variable         |                     Value(s) supported                         |
|:--------------------------------|:---------------------------------------------------------------|
|   CANVAS_LMS_ADMIN_EMAIL        | E-mail address used for default administrator login            |
|   CANVAS_LMS_ADMIN_PASSWORD     | Password for default administrator login                       |
|   CANVAS_LMS_ACCOUNT_NAME       | Account name seen by users, usually your organization name     |
|   CANVAS_LMS_STATS_COLLECTION   | opt_in, opt_out, or anonymized                                 |


## Canvas ownership
### Making sure Canvas can't write to more things than it should.

    root@server:/$  chown -R root:root /var/canvas
    root@server:/$  cd /var/canvas
    root@server:/var/canvas$  chown -R www-canvas config/environment.rb log tmp public/assets \
                              app/stylesheets/_brandable_variables_defaults_autogenerated.scss \
                              app/stylesheets/brandable_css_brands Gemfile.lock config.ru

### Making sure other users can't read private Canvas files

There are a number of files in your configuration directory (/var/canvas/config) that contain passwords, encryption keys, and other private data that would compromise the security of your Canvas installation if it became public. These are the .yml files inside the config directory, and we want to make them readable only by the canvasuser user.

    root@server:/var/canvas$  chown www-canvas config/*.yml
    root@server:/var/canvas$  chmod 400 config/*.yml

Note that once you change these settings, to modify the configuration files henceforth, you will have to use `sudo`.


## Making sure to use the "most restrictive" permissions

Passenger will choose the user to run the application based on the **ownership** settings of `config/environment.rb` (you can view the ownership settings via the ls -l command). Note that it is probably wise to ensure that the ownership settings of all other files besides the ones with permissions set just above are restrictive, and only allow your canvasuser user account to read the rest of the files.


## Apache configuration
### Installation

You're now going to need to set up the webserver. We're going to use Apache and Passenger to serve the Canvas content. Before proceeding, you'll need to add the Phusion Passenger APT repository, which contains the passenger package. After you've installed the repository, you'll need to install the apache and passenger packages. If you are on Debian/Ubuntu, you can do this quickly by typing:

    root@server:/$  apt-get install passenger libapache2-mod-passenger apache2

We'll be using mod_rewrite, so you'll want to enable that.

    root@server:/$  a2enmod rewrite
    root@server:/$  a2enmod passenger

and follow the instructions.

Once you have Apache and Passenger installed, we're going to need to set up Apache, Passenger, and your Rails app to all know about each other. This will be a brief overview, and for more detail, you should check out the Passenger documentation for setting up Apache.


### Configure Passenger with Apache

First, make sure Passenger is enabled for your Apache configuration. In Debian/Ubuntu, the libapache2-mod-passenger package should have put symlinks inside of /etc/apache2/mods-enabled/ called passenger.conf and passenger.load. If it didn't or they are disabled somehow, you can enable passenger by running:

    root@server:/$  a2enmod passenger

In other setups, you just need to make sure you add the following lines to your Apache configuration, changing paths to appropriate values if necessary:

    LoadModule passenger_module /usr/lib/apache2/modules/mod_passenger.so
    PassengerRoot /usr
    PassengerRuby /usr/bin/ruby

If you have trouble starting the application because of permissions problems, you might need to add this line to your passenger.conf, site configuration file, or httpd.conf (where canvasuser is the user that Canvas runs as, or www-canvas as above):

    PassengerDefaultUser www-canvas

### Configure SSL with Apache

Next, we need to make sure your Apache configuration supports SSL. Debian/Ubuntu doesn't ship Apache with the SSL module enabled by default, so you will need to create the appropriate symlinks to enable it.

    root@server:/$  a2enmod  ssl

 You can use **Let's Encrypt** with Apache2 (**mod_md**)
    

    
### Configure Canvas with Apache


Now, we need to make a VirtualHost for your app. On Debian/Ubuntu, we are going to need to make a new file called /etc/apache2/sites-available/canvas. On other setups, find where you put VirtualHosts definitions. You can open this file like so:

    root@server:/$  nano /etc/apache2/sites-available/canvas.conf

In the new file, or new spot, depending, you want to place the following snippet. You will want to modify the lines designated ServerName(2), ServerAdmin(2), DocumentRoot(2), SetEnv(2), Directory(2), and probably SSLCertificateFile(1) and SSLCertificateKeyFile(1), discussed below in the "Note about SSL Certificates".

    <VirtualHost *:80>
      ServerName canvas.example.com
      ServerAlias rce.canvas.example.com canvasfiles.example.com
      ServerAdmin youremail@example.com
      
      DocumentRoot /var/www/html
      
      RewriteEngine On
      RewriteCond %{HTTP:X-Forwarded-Proto} !=https
      RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [L]
      
        ErrorLog ${APACHE_LOG_DIR}/canvas_error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/canvas_access.log combined
    </VirtualHost>
    
    
        
    <VirtualHost *:443>
      ServerName canvas.example.com
      ServerAlias canvasfiles.example.com
      ServerAdmin youremail@example.com
      
      DocumentRoot /var/canvas/public
      
      PassengerUser www-canvas
    #  SetEnv RAILS_ENV production
      PassengerAppEnv production
      PassengerStartTimeout 300
    #  PassengerAppRoot /var/canvas
    #  PassengerAppType rack
    #  PassengerStartupFile config.ru
    #  PassengerMinInstances 1
      
      <Directory /var/canvas/public>
        Options All
        AllowOverride All
        Require all granted
     </Directory>
      
      SSLEngine on
      BrowserMatch "MSIE [17-9]" ssl-unclean-shutdown
      # the following ssl certificate files are generated for you from the ssl-cert package.
      SSLCertificateFile /etc/ssl/certs/ssl-cert-snakeoil.pem
      SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
      
        ErrorLog ${APACHE_LOG_DIR}/canvas_error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/canvas_access.log combined
    </VirtualHost>

    
Apache <2.4 users: the allow/options configuration inside the <Directory /var/canvas/public> have changed in Apache 2.4. In older versions of Apache, you'll likely want something like this:

      <Directory /var/canvas/public>
        Allow from all
        Options -MultiViews
      </Directory>
     
     
And finally, if you created this as its own file inside /etc/apache2/sites-available, we'll need to make it an enabled site.

    root@server:/$ sudo a2ensite canvas

    


## Automated jobs

Canvas has some automated jobs that need to run at occasional intervals, such as email reports, statistics gathering, and a few other things. Your Canvas installation will not function properly without support for automated jobs, so we'll need to set that up as well.

Canvas comes with a daemon process that will monitor and manage any automated jobs that need to happen. If your application root is /var/canvas, this daemon process manager can be found at /var/canvas/script/canvas_init.

**You'll need to run these job daemons on at least one server.** Canvas supports running the background jobs on multiple servers for capacity/redundancy, as well.

Because Canvas has so many jobs to run, it is advisable to dedicate one of your app servers to be just a job server. You can do this by simply skipping the Apache steps on one of your app servers, and then only on that server follow these automated jobs setup instructions.
Installation

If you're on Debian/Ubuntu, you can install this daemon process very easily, first by making a symlink from /var/canvas/script/canvas_init to /etc/init.d/canvas_init, and then by configuring this script to run at valid runlevels (we'll be making an upstart script soon):

    root@server:/$  ln -s /var/canvas/script/canvas_init /etc/init.d/canvas_init
    root@server:/$  update-rc.d canvas_init defaults
    root@server:/$  /etc/init.d/canvas_init start
 
 OR on systemd:  
    root@server:/$ systemctl start canvas_init 

### NOTE:  
On Debian10 systems, the canvas_init start is may failing, if the `www-canvas` UID is less than 1000 (if it is a system user), due to the missing shell environment.  
So you need to edit the init script, and add the following manually:  

change the row:   
`exec su $(stat -c %U $(dirname $(readlink -f $0))/../config/environment.rb) -c "/bin/bash $0 $@"`  
to:  
`exec su $(stat -c %U $(dirname $(readlink -f $0))/../config/environment.rb) -s /bin/bash -c "/bin/bash $0 $@"`  


[More info](https://github.com/instructure/canvas-lms/issues/1693)  



    
## Optimizing File Downloads

If you are storing uploaded files locally, rather than in S3, you can optimize the downloading of files using the X-Sendfile header (X-Accel-Redirect in nginx). First make sure that apache has mod_xsendfile installed and enabled. For UBUNTU this can be done by following command:

    sysadmin@appserver:/var/canvas$ sudo apt-get install libapache2-mod-xsendfile

This command installs and enables module. To ensure about properly running module you can use:

    sysadmin@appserver:/var/canvas$ sudo apachectl -M | sort

Module xsendfile_module (shared) should be in the list.

In `config/environments/production.rb` you'll find the necessary `config.action_dispatch.x_sendfile_header` line, but commented out. We recommend that you create a `config/environments/production-local.rb` file and add the uncommented line to that file, to avoid future merge conflicts.

In your canvas virtual host at `/etc/apache2/sites-available/canvas.conf` , add the following two directives:

    XSendFile On
    XSendFilePath /var/canvas
    
### NOTE  

If your server runs more other websites, **check carefully the effect of xsendfile** for the other services! Because, sometimes, other Cloud web services does not accepts the xsendfile module enabled.  
    
    
## Cache configuration

Canvas supports two different methods of caching: Memcache and redis. However, there are some features of Canvas that require redis to use, such as OAuth2, so it's recommended that you use redis for caching as well to keep things simple.  

Below are instructions for setting up redis.  
[Canvas Wiki](https://github.com/instructure/canvas-lms/wiki/Production-Start)





## QTIMigrationTool

The QTIMigrationTool needs to be installed to enable copying or importing quiz content. Instructions are at https://github.com/instructure/QTIMigrationTool/wiki. After installation, ensure the plugin is active in Site Admin -> Plugins -> QTI Converter (it should detect the QTIMigrationTool and auto-activate).

### INSTALL  

First check if, python 2.x is installed:  

    root@server:/$  python --version  

The **python-lxml** package is needed:  

    root@server:/$  apt-get install python-lxml  

Then clone the git repo, copy or move files, set the permissions, etc.  

    root@server:/$  cd /var/canvas/vendor
    root@server:/$  git clone https://github.com/instructure/QTIMigrationTool.git QTIMigrationTool
    root@server:/$  chown -R root:root QTIMigrationTool/
    root@server:/$  cd QTIMigrationTool && chmod +x migrate.py
    
Finally: restart the background job manager with:
    
    root@server:/$  /etc/init.d/canvas_init restart






## Rich Content Editor

Canvas includes a new rich content editor component to support a consistent editor experience across multiple applications in the Canvas ecosystem. To make use of this component you need to run a supporting API server. See the [Canvas RCE API Documentation](https://github.com/instructure/canvas-rce-api/blob/master/README.md) for information on running the service and configuring Canvas to make use of it. Starting July 14, 2018, the stable branch of canvas-lms will require this service to be running and configured for full rich content editing functionality.

### INSTALL  

The application can be run directly with Node.js by either running `npm start` or `node app.js`.  
It is designed to work with the current Node.js LTS (10.x) release.  

    root@server:/$  cd /var/  
    root@server:/$  git clone https://github.com/instructure/canvas-rce-api.git  

First install all of the package dependencies:

    root@server:/$  cd /var/canvas-rce-api
    root@server:/$  npm upgrade
    root@server:/$  npm install --production

Create the `.env` configuration file.  
    
    cp .env.example .env  

Edit the file: apply your changes:  
More configuration variables are [here](https://github.com/instructure/canvas-rce-api)  

    PORT=3000
    NODE_ENV=production
    #STATSD_HOST=127.0.0.1
    #STATSD_PORT=8125
    #STATS_PREFIX=rceapi
    ECOSYSTEM_SECRET="astringthatisactually32byteslong"
    ECOSYSTEM_KEY="astringthatisactually32byteslong"
    #CIPHER_PASSWORD=TEMP_PASSWORD

On the Canvas-LMS configuration side: Edit the `dynamic_settings.yml` file with the following  

    production:
      config:
        canvas:
          canvas:
            encryption-secret: "astringthatisactually32byteslong"
            signing-secret: "astringthatisactually32byteslong"
          rich-content-service:
            app-host: "https://canvas.example.com/rce"
        
Note: The bytestring should match on both side:  

    ECOSYSTEM_SECRET = signing-secret
    ECOSYSTEM_KEY = encryption-secret

## Set The Rights

    root@server:/$  chown -R root:root /var/canvas-rce-api
    root@server:/$  chown www-canvas /var/canvas-rce-api/.env
    root@server:/$  chmod 400 /var/canvas-rce-api/.env  


## RUNNING

Run the node application in the background:  
NOTE: The instance **Should Not Run with root privileges!**

    root@server:/$  sudo -u www-canvas npm start

Edit the apache2 configuration file, `/etc/apache2/sites-available/canvas.conf` and add the following: (within the wirtualhost)

    SSLProxyEngine On
    ProxyPass /rce http://localhost:3000 retry=1 acquire=3000 timeout=600 keepalive=On

Enable the apache2 proxy modules, and restart the webserver:

    root@server:/$  a2enmod proxy
    root@server:/$  a2enmod proxy_http
    root@server:/$  /etc/init.d/apache2 restart
    
    
    
## RUNNING (A Better Alternative)

If you have a "single-server" environment, you can run the **RCE API**, and the **Canvas-LMS** together on the same apache2 server.

Create a secondary domain for rce api, eg: `rce.canvas.example.com` and configure a apache2 site for it:

    root@server:/$  nano /etc/apache2/sites-available/canvas_rce.conf

Add the following content to the end of the config file:  
(After the last existing <VirtualHost> configuration)  

    <VirtualHost *:443>
      ServerName rce.canvas.example.com
      ServerAlias rce.canvasfiles.example.com
      ServerAdmin youremail@example.com
      
      DocumentRoot /var/www/html
      
      PassengerUser www-canvas
      PassengerAppRoot /var/canvas-rce-api
      PassengerAppType node
      PassengerStartupFile app.js
      PassengerStartTimeout 30
      PassengerAppEnv production
    #  PassengerMinInstances 1
      
      SSLEngine on
      BrowserMatch "MSIE [17-9]" ssl-unclean-shutdown
      # the following ssl certificate files are generated for you from the ssl-cert package.
      SSLCertificateFile /etc/ssl/certs/ssl-cert-snakeoil.pem
      SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
      
        ErrorLog ${APACHE_LOG_DIR}/canvas_rce_error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/canvas_rce_access.log combined
    </VirtualHost>


In this case, the Canvas-LMS RCE should be configured in `dynamic_settings.yml` as:

          rich-content-service:
            app-host: "https://rce.canvas.example.com"

And restart apache2

    root@server:/$  /etc/init.d/apache2 restart





## Ready, set, go!
Restart Apache (sudo /etc/init.d/apache2 restart), and point your browser to your new Canvas installation! Log in with the administrator credentials you set up during database configuration, and you should be ready to use Canvas.

## Troubleshooting
We have a full page of frequently asked questions about troubleshooting your Canvas installation. See our Troubleshooting page.

## Common configuration options
There are many other aspects of Canvas that you can now configure, having a working production environment. Please see Canvas Integration for more information.




### list installed NPM modules
    npm list
    npm -g list
