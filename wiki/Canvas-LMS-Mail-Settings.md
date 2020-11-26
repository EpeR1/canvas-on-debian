# Outgoing Mail 

Outgoing mail settings are in the `/var/canvas/config/outgoing_mail.yml`, (where the /var/canvas is your canvas-root)  
You can choose **SMTP over SSL**, or **SMTP over STARTTLS** or **Sendmail/Postfix-pickup** modes for email delivery service.  
The **domain** field is required.
The **outgoing_address** field is (strongly) recommended. 

An example for **Sendmail:**  

    production:
      delivery_method: "sendmail"
      domain: "canvas.example.com"
      outgoing_address: "canvas@canvas.example.com"
      default_name: "Your Canvas Application"

For **SMTP over SSL:**  

    production:
      delivery_method: "smtp"
      enable_starttls_auto: "false"
      openssl_verify_mode: "none"  # or peer
      tls: "false"
      address: "smtp.server.example.com"
      user_name: "smtp-user-name"
      password: "smtp-user-password"
      authentication: "plain" # plain, login, or cram_md5
      port: "465"
      domain: "canvas.example.com"
      outgoing_address: "canvas@canvas.example.com"
      default_name: "Your Canvas Application"



For **SMTP over STARTTLS:**

    production:
      delivery_method: "smtp"
      enable_starttls_auto: "true"
      openssl_verify_mode: "none"  # or peer
      tls: "true"
      address: "smtp.server.example.com"
      user_name: "smtp-user-name"
      password: "smtp-user-password"
      authentication: "plain" # plain, login, or cram_md5
      port: "587"
      domain: "canvas.example.com"
      outgoing_address: "canvas@canvas.example.com"
      default_name: "Your Canvas Application"



## NOTE

On the **Site Admin** Settings, set, the **Course level overrides for notification preferences [hidden]** feature to **OFF** !

More info about Rails Action Mailer are [here](https://guides.rubyonrails.org/action_mailer_basics.html)

---------------------------------------------
# Incoming Mail 

Outgoing mail settings are in the `/var/canvas/config/incoming_mail.yml`, (where the /var/canvas is your canvas-root)  
You can choose **File**, or **IMAP** or **POP3** modes for email source.

An example for **FILE**

    production:
      run_periodically: true
      directory:
        folder: "/path/to/maildir"   #or "../mail/new" If your canvas-user homedir is: "/var/canvas", and maildir is: "$HOME/mail"


For **IMAP**

    production:
      run_periodically: true
      imap:
        server: "imap.server.example.com"
        port: 993
        ssl: true
        filter: ALL
        username: "imap-user"
        password: "imap-user-password"


For **POP3**

    production:
      run_periodically: true
      imap:
        server: "pop3.server.example.com"
        port: 995
        ssl: true
        filter: ALL
        username: "pop3-user"
        password: "pop3-user-password"


## NOTE 1
If your server are running a Postfix service along with Canvas instance, you can choose the **file source**, for incoming, and the **Sendmail** for outgoing mail service, when your Postfix is configured with `home_mailbox = mail/` maildir storage format.

## NOTE 2
Don't forget to **restart** the **Apache2 Passenger** and the **Canvas Background Job** service, after the config modification!

    /etc/init.d/apache2 restart 
    /etc/init.d/canvas_init stop 
    /etc/init.d/canvas_init start
