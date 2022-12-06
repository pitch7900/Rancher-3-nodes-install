**Configuration apache2 en reverse proxy**
======================================

````bash
apt update
apt -y upgrade
apt -y install apache2 certbot python3-certbot-apache
a2dissite 000-default
a2enmod ssl rewrite headers proxy deflate env proxy_http negociation negotiation status proxy_balancer substitute
systemctl restart apache2
````

File :  /etc/apache2/sites-available/http.rancher.mydomain.com.conf

````apacheconf
<VirtualHost _default_:80>
  ProxyPreservehost Off
  ServerName  rancher.mydomain.com
  ServerAlias rancher.mydomain.com

  AllowEncodedSlashes NoDecode
  CustomLog ${APACHE_LOG_DIR}/access.rancher.mydomain.com_80.log combined

  <Directory /var/www/rancher>
    AllowOverride All
  </Directory>
  ErrorLog ${APACHE_LOG_DIR}/error.rancher.mydomain.com_80.log
  # Possible values include: debug, info, notice, warn, error, crit,
  # alert, emerg.
  LogLevel warn
  CustomLog ${APACHE_LOG_DIR}/access.rancher.mydomain.com_80.log combined
  DocumentRoot /var/www/rancher

RewriteEngine on
RewriteCond %{SERVER_NAME} =rancher.mydomain.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
````

````bash
mkdir -p /var/www/rancher/.well-known
echo "Options +FollowSymLinks
RewriteEngine on
RewriteCond %{REQUEST_URI} !^/.well-known/

RewriteRule (.*) https://rancher.mydomain.com/$1 [R=301,L]
">/var/www/rancher/.htaccess
chown -R www-data:www-data /var/www/
chmod 644 /var/www/rancher/.htaccess
a2ensite http.rancher.mydomain.com
systemctl reload apache2
certbot --apapche
a2dissite http.rancher.mydomain.com-le-ssl
````

File :  /etc/apache2/sites-available/ssl.rancher.mydomain.com.conf

````apacheconf
<IfModule mod_ssl.c>
<VirtualHost _default_:443>
  SSLEngine On
  SSLProxyEngine on
  SSLProxyVerify none 
  SSLProxyCheckPeerCN off
  SSLProxyCheckPeerName off


  SSLCertificateFile /etc/letsencrypt/live/rancher.mydomain.com/fullchain.pem
  SSLCertificateKeyFile /etc/letsencrypt/live/rancher.mydomain.com/privkey.pem
  Include /etc/letsencrypt/options-ssl-apache.conf
 
  RequestHeader set "X-Forwarded-Proto" expr=%{REQUEST_SCHEME}
  RequestHeader set "X-Forwarded-SSL" expr=%{HTTPS}
  RequestHeader set X-Forwarded-Port "443"
  RequestHeader set Host "rancher.mydomain.com"
  #Header edit Location "(^http[s]?://)([^/]+)" ""
  ProxyRequests Off
  ProxyPreserveHost On
  ProxyPassReverseCookieDomain kube.mydomain.local rancher.mydomain.com

  ServerName  rancher.mydomain.com
  ServerAlias rancher.mydomain.com

  ErrorLog ${APACHE_LOG_DIR}/error.rancher.mydomain.com.log
  # Possible values include: debug, info, notice, warn, error, crit,
  # alert, emerg.
  LogLevel warn
  #CustomLog ${APACHE_LOG_DIR}/access.rancher.mydomain.com.log combined

  CustomLog ${APACHE_LOG_DIR}/access.rancher.mydomain.com.log "%h %l %{SSL_CLIENT_S_DN}x  %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-agent}i\""

  <Proxy "balancer://mycluster">
    BalancerMember "https://10.79.1.201:443"
    BalancerMember "https://10.79.1.202:443"
    BalancerMember "https://10.79.1.203:443"
  </Proxy>
  <Location />
    ProxyPass "balancer://mycluster/"
    ProxyPassReverse "balancer://mycluster/"
  </Location>
  
</VirtualHost>
</IfModule>
````

````bash
a2ensite ssl.rancher.mydomain.com
systemctl reload apache2
````

[**<<Retour**][Home]

[Home]: /README.md
