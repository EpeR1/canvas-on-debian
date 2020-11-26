
### This is an unofficially upgraded installation guide for [Canvas LMS](https://github.com/instructure/canvas-lms), on [Debian 10](https://www.debian.org/News/2019/20190706).  

Canvas includes a new rich content editor component to support a consistent editor experience across multiple applications in the Canvas ecosystem.  
To make use of this component you need to run a supporting API server.  
See the [Canvas RCE API Documentation](https://github.com/instructure/canvas-rce-api/blob/master/README.md) for information on running the service and configuring Canvas to make use of it.  
Starting July 14, 2018, the stable branch of canvas-lms will require this service to be running and configured for full rich content editing functionality.

## INSTALL  

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


## RUNNING 1.

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
    
    
    
## RUNNING 2. (Better Alternative)

If you have a "single-server" environment, you can run the [RCE API](https://github.com/instructure/canvas-rce-api), and the [Canvas LMS](https://github.com/instructure/canvas-lms) together on the same apache2 server.

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



