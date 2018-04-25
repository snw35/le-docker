# le-docker
Letsencrypt docker compose.

This docker-compose file will automate the deployment of my letsencrypt images:

 * [le-certbot](https://github.com/snw35/le-certbot) - [![Build Status](https://travis-ci.org/snw35/le-certbot.svg?branch=master)](https://travis-ci.org/snw35/le-certbot)
 * [le-nginx](https://github.com/snw35/le-nginx) - [![Build Status](https://travis-ci.org/snw35/le-nginx.svg?branch=master)](https://travis-ci.org/snw35/le-nginx)

You can use this to automatically generate and renew letsencrypt certificates for any number of domains or subdomains, and to securely proxy traffic to your applications with these certificates, all without bind-mounting the docker-socket into any containers. (See [here]((https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface_) for why you don't want to do that.)

## How to use

 1. Install docker and docker-compose if needed.
 1. Make sure that port 80 and 443 are available on the machine/VM.
 1. Make sure that your domain(s) resolve to the correct IP address.
 1. Make sure that port 80 and 443 are forwarded through your router/firewall to the machine, and that no firewall on the machine itself is blocking them.
 1. Create the two needed volumes, the backplane network, and clone the repo:
```
docker volume create le-certs
docker volume create le-conf
docker network create le-backplane
git clone https://github.com/snw35/le-docker
cd le-docker
```
 1. Edit the docker-compose.yml file and enter the first domain you would like to generate a certificate for against LE_DOMAIN_1. If you have more, add them as LE_DOMAIN_2, LE_DOMAIN_3, etc. Add your email address against LE_EMAIL, then bring the environment up:
```
docker-compose up -d
```

 1. The le-certbot container will attempt to generate a new certificate for each of the configured domains if none already exist. Check the log to see its progress:
```
docker-compose logs
```
    See the troubleshooting section below if anything went wrong.

 1. Connect your application(s) to the 'le-backplane' network. If your application has several containers, only the "front-end" one needs to connect to it, e.g the webserver or endpoint. So for example, in a nextcloud deployment, you would conect the nginx or apache container serving nextcloud to the 'le-backplane' network.

 1. Place an nginx SSL proxy configuration file for your application into the le-conf volume (see nginx section below for an example template file). This can be done with a temporary intermediate container to access the volume through docker like so:
 ```
 docker run -dit --rm --mount source=le-conf,target=/etc/nginx/ alpine:3.7 sh
 cd /etc/nginx/conf.d
 ```
When you are done, restart the compose environment to make sure nginx starts with the new configuration:
 ```
 docker-compose down
 docker-compose up -d
 docker-compose logs
 ```

 1. If all containers are still running and there are no errors in the logs, you should now be able to access your application at https://your.domain

## Where are my certificates?

They are stored inside the le-certs volume. You can find them in:
```
/var/lib/docker/volumes/le-certs/_data/
```
Mounting this volume inside a container will give that container access to your certificates, e.g:
```
docker run -d --mount source=le-certs,target=/etc/letsencrypt myimage:latest
```

## How do I configure nginx to proxy $app?

Some good resources for creating nginx config files are:

 * Mozilla's SSL conf generator: https://mozilla.github.io/server-side-tls/ssl-config-generator/
 * Config generator by valentinxxx: https://nginxconfig.io/

Here is the template that I am using, which is based on Mozilla's generator
(the parts you need to edit are enclosed in \_\_<>\_\_):

```
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    # certs sent to the client in SERVER HELLO are concatenated in ssl_certificate
    ssl_certificate /etc/letsencrypt/live/__<your domain>__/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/__<your domain>__/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;


    # modern configuration. tweak to your needs.
    ssl_protocols TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
    ssl_prefer_server_ciphers on;

    # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
    add_header Strict-Transport-Security max-age=15768000;

    # OCSP Stapling ---
    # fetch OCSP records from URL in ssl_certificate and cache them
    ssl_stapling on;
    ssl_stapling_verify on;

    ## verify chain of trust of OCSP response using Root CA and Intermediate certs
    ssl_trusted_certificate /etc/letsencrypt/live/__<your domain>__/fullchain.pem;

    resolver __<your DNS server>__;

    location / {

      proxy_set_header        Host $host;
      proxy_set_header        X-Real-IP $remote_addr;
      proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header        X-Forwarded-Proto $scheme;

      proxy_pass          __<your application URL, e.g http://container:8080>__;

    }
}

```


## Troubleshooting

If something went wrong, then the le-certbot container will exit with a non-zero return code and more information should be available in the logs:
```
docker-compose logs
```

__NOTE:__ While troubleshooting, edit the docker-compose.yml file and put 'y' next to the LE_TEST environment variable. This will tell Certbot to request test certificates, which don't count towards your limited allocation of genuine certificate requests. This means you can restart the environment as many times as you like without locking your letsencrypt account out for a period of time.


See https://letsencrypt.org/docs/rate-limits/ for more information.


You should be able to see the nginx webserver receiving the challenge requests from the letsencrypt acme challenge server in the docker-compose logs. If you can't see these, then incorrect DNS, port forwards, or a local firewall rules are the most likely culprits.

Beyond this, much of the troubleshooting steps from the Certbot website apply:

https://certbot.eff.org/docs/using.html

If you are still having trouble, I would recommend installing certbot natively on a test system and seeing if you can generate a single test certificate that way.

### Running on an Alpine host

__Note__ that certbot docker containers currently cannot run on an Alpine Linux host due to an [unresolved bug/issue](https://github.com/certbot/certbot/issues/5737). (This is Alpine as the main OS on which Docker is installed, not just as the base of the image).
