## Why Client-Side Certificate Authentication?  Why nginx? ##

I sometimes peruse the ReST questions of stackoverflow.com.  Many times I see questions about authentication.  There are many options (Basic HTTP Auth, Digest HTTP Auth, OAuth, [OAuth Wrap](http://wiki.oauth.net/OAuth-WRAP), etc.) however when security is of importance, I like to recommend client side certificates.  This is the route our team at ShowClix chose when implementing our API.

When first implementing the API Authentication, we were using Apache for our ReST API Servers.  It took some serious google-fu and tinkering to get the Apache cooperating with the client-side certs and passing that info into our PHP App layer.  I remember it being a semi-painful process.

Lately, I've become a huge fan of [nginx](http://wiki.nginx.org/Main).  Its clean, familiar config syntax and speed make it a great alternative for Apache in many cases.  Its [reverse proxy](http://wiki.nginx.org/NginxHttpProxyModule) capabilities are quite nice as well.  So, I thought I'd give client-side cert authentication a shot in nginx.  Whereas a quick search for "Client Side Certs in Apache" yielded a few relevant results, a similar search for nginx yielded no results, so I figured I'd share here.

**I ran this on a small 256MB [Rackspace cloudserver](http://www.rackspacecloud.com/cloud_hosting_products/servers) instance running [Arch Linux](http://www.archlinux.org/), nginx 0.7.65, PHP 5.3.2 and PHP FPM.**

## Creating and Signing Your Certs ##

This is SSL, so you'll need an cert-key pair for you/the server, the api users/the client, and a CA pair.  You will be the CA in this case (usually a role played by VeriSign, thawte, GoDaddy, etc.), signing your client's certs. There are [plenty of tutorials out there](http://www.tc.umn.edu/~brams006/selfsign.html) on creating and signing certificates, so I'll leave the details on this to someone else and just quickly show a sample here to give a complete tutorial. *NOTE: This is just a quick sample of creating certs and not intended for production.*

    
    # Create the CA Key and Certificate for signing Client Certs
    openssl genrsa -des3 -out ca.key 4096
    openssl req -new -x509 -days 365 -key ca.key -out ca.crt
 
    # Create the Server Key, CSR, and Certificate
    openssl genrsa -des3 -out server.key 1024
    openssl req -new -key server.key -out server.csr
 
    # We're self signing our own server cert here.  This is a no-no in production.
    openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt
    
    # Create the Client Key and CSR
    openssl genrsa -des3 -out client.key 1024
    openssl req -new -key client.key -out client.csr
    
    # Sign the client certificate with our CA cert.  Unlike signing our own server cert, this is what we want to do.
    openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out client.crt


## Configuring nginx ##

    server {
        listen        443;
        ssl on;
        server_name example.com;
     
        ssl_certificate      /etc/nginx/certs/server.crt;
        ssl_certificate_key  /etc/nginx/certs/server.key;
        ssl_client_certificate /etc/nginx/certs/ca.crt;
        ssl_verify_client optional;
     
        location / {
            root           /var/www/example.com/html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_param  SCRIPT_FILENAME /var/www/example.com/lib/Request.class.php;
            fastcgi_param  VERIFIED $ssl_client_verify;
            fastcgi_param  DN $ssl_client_s_dn;
            include        fastcgi_params;
        }
    }


The main things to note here are...

 - We specify our the server's certificate (`server.crt`) and private key (`server.key`)
 - We specify the CA cert that we used to sign our client certificates (`ca.crt`)
 - We set the `ssl_verify_client` to `optional`.  This tells nginx to attempt to verify to SSL certificate if provided.  My API allows both authenticated and authenticated requests, however if you only want to allow authenticated requests, you can go ahead and set this value to `on`.
 - Lastly, you'll notice that I add a `location` directive that routes all the requests to a single PHP script.  You can handle this differently (and technically don't even need to use PHP as there are other fast cgi options)

## Passing to PHP ##

There are several options for running PHP from nginx.  I chose to use [PHP FPM](http://php-fpm.org/), however these steps should also work for any of the fast cgi options in theory.  You'll notice I added a few additional `fastcgi_params` to the usual fastcgi_params.

First, we pass in the `$ssl_client_verify` variable as the `VERIFIED` parameter.  This is useful when we are allowing authenticated and unauthenticated requests.  When the client certificate was able to be verified against our CA cert, this will have the value of `SUCCESS`. Otherwise, the value will be `NONE`.

Second, you'll notice we pass the `$ssl_client_s_dn` variable to the `DN` parameter.  This will provide "the line subject DN of client certificate for established SSL-connection".  The Common Name part of this certificate may be of most interest for you.  Here is an example value for DN...

    /C=US/ST=Florida/L=Orlando/O=CLIENT NAME/CN=CLIENT NAME

Nginx also provides the option to pass in the entire client certificate via `$ssl_client_cert` or `$ssl_client_cert_raw`.  For more details on the SSL options available to you in nginx, checkout the [Nginx Http SSL Module Wiki](http://wiki.nginx.org/NginxHttpSslModule).

## Consuming the ReST Service ##

So, we've created our certs, signed our client certs, installed nginx and PHP, and setup nginx verify the certs and finally pass along client cert details.  Now we are ready to consume our ReSTful service.

There are lots of tools out there for consuming true HTTP based ReSTful services.  Some of them are [prettier](http://code.google.com/p/rest-client/) than others, but I prefer the good old cli version of [cURL](http://curl.haxx.se/). 

*NOTE: This is run from your client computer.  Make certain that you have `scp`d your client.key and client.crt files from your server onto your client machine that is making the requests.  You'll also be prompted for the pass phrase you used when you first created the client cert.  There are ways to [remove the need for pass phrases](http://www.insivia.com/blog/removing-a-pass-phrase-from-a-ssl-certificate/).  Also, you don't need the verbose flag (-v) or silent flags (-s).  We're using the -k flag here because we used a self-signed cert for the server.*

    curl -v -s -k --key client.key --cert client.crt https://example.com

