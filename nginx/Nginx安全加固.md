# Nginx server security - hardening Nginx configuration

## Disable server_tokens Directive
Web server 404 pages will typically display some bits of privileged information—in Nginx’s case, turning server_tokens off will prevent it from displaying the web server version being used.
```
http{
    server_tokens off;
}
```

## Disable unwanted HTTP methods
This can be set in the “server” section of the Nginx configuration. By adding a condition to allow only GET/HEAD/POST methods,less-inocuous methods like TRACE and DELETE are met with a 444 No Response status code.
```
if ($request_method !~ ^(GET|HEAD|POST)$ ){
       return 444;
}
```

## Disable weak cipher suites
Weak cipher suites may lead to vulnerability like a logjam, and that’s why we need to allow only strong cipher.
```
ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 
EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 
EECDH EDH+aRSA HIGH !RC4 !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS";
```

## Redirect HTTP traffic to HTTPS.
Forcing all connections to use encryption can reduce the chance of snooping and man-in-the-middle (MITM) attacks. Using the directive return 301 https://$server_name$request_uri will set up permanent URL redirection for all requests made to port 80.
```
return 301 https://$server_name$request_uri;
```

## Limit the number of connections permitted per IP address
The ngx_http_limit_conn_module module is used to limit the number of connections per the defined key, in particular, the number of connections from a single IP address.
```
limit_conn_zone $binary_remote_addr zone=perip:10m;
limit_conn_zone $server_name zone=perserver:10m;

server {
    ...
    limit_conn perip 10;
    limit_conn perserver 100;
}
```

## Clickjacking Attack
By configuring Nginx to use the X-Frame-Options header with the value “SAMEORIGIN,” browsers rendering pages inside a <frame> or <iframe> will not be as easily subjected to clickjacking attacks.
```
server {
    add_header X-Frame-Options "SAMEORIGIN";
}
```

## Buffer Overflow Attack
Buffer overflow attacks are made possible by writing data to a buffer and exceeding that buffers’ boundary and overwriting memory fragments of a process. To prevent this in nginx we can set buffer size limitations for all clients. This can be done through the Nginx configuration file using the following directives:
```
client_body_buffer_size 1K; 
client_header_buffer_size 1k; 
client_max_body_size 1k; 
large_client_header_buffers 2 1k;
```

## X-XSS Protection
When add_header X-XSS-Protection “1; mode=block”; is included in your configuration file, Nginx will add X-XSS protection to headers to mitigate XSS and clickjacking attacks.
```
add_header X-XSS-Protection "1; mode=block";
```

## Keep Nginx up to date
Last but not least, you need to keep your nginx up-to-date as there are many performance enhancement, security fixes and new features are being added.
```
#CentOS based systems
yum update nginx
#Ubuntu and Debian
apt-get upgrade nginx
```