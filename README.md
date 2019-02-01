# le-docker
Letsencrypt docker compose.

This docker-compose file will automate the deployment of:

 * [le-certbot](https://github.com/snw35/le-certbot) - [![Build Status](https://travis-ci.org/snw35/le-certbot.svg?branch=master)](https://travis-ci.org/snw35/le-certbot)
 * [le-nginx](https://github.com/snw35/le-nginx) - [![Build Status](https://travis-ci.org/snw35/le-nginx.svg?branch=master)](https://travis-ci.org/snw35/le-nginx)

Deployed as a frontend SSL proxy service, they will:

 * Automate the generation and renewal of letsencrypt SSL certificates.
 * Provide strong security (A+ rating on [SSL Labs](https://www.ssllabs.com)).
 * Not expose the docker socket in any way, making them suitable for production deployment.

You can use this docker compose file to automatically generate and renew letsencrypt certificates for any number of domains. This solution does not require bind-mounting the docker daemon socked into a running container, which the Docker [documentation on security](https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface) does not reccomend for production environments.

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

 1. Edit the docker-compose.yml file and enter the first domain you would like to generate a certificate for against LE_DOMAIN_1. If you have more domains or subdomains, add them as LE_DOMAIN_2, LE_DOMAIN_3, etc. Add your email address against LE_EMAIL, then bring the environment up:

    ```
    docker-compose up
    ```

    The le-certbot container will attempt to generate a new certificate for each of the configured domains if none already exist. See the troubleshooting section below if anything went wrong.
    Press Ctrl-Z and then type 'bg' to background the docker-compose process.

 1. Connect your application(s) to the 'le-backplane' network. If your application has several containers, only the "front-end" one needs to connect to it, e.g the webserver or endpoint.

 1. Place an nginx SSL proxy configuration file for your application into the le-conf volume (see nginx section below for an example template file). This can be done by entering the le-nginx container like so:

    ```
    docker-compose exec le-nginx sh
    cd /etc/nginx/conf.d/
    vi ./newconffile.conf
    ```

    When you are done, recreate the le-nginx service to use the new configuration:

    ```
    docker-compose up le-nginx
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
Or, in docker-compose:
```
    service:
        volumes:
            - le-certs:/etc/letsencrypt
```

## How do I configure nginx to proxy $app?

Some good resources for creating nginx config files are:

 * Mozilla's SSL conf generator: https://mozilla.github.io/server-side-tls/ssl-config-generator/
 * Config generator by valentinxxx: https://nginxconfig.io/

Here is an example template based on Mozilla's generator and SSL Labs' recommendations for cyphers to use. The variables that you will need to replace are:
 * $domain - the fully qualified domain name this service should respond to ( e.g sub.example.com)
 * $container - the name of the container the proxy should connect back to. This will be the webserver or frontend if there are multiple containers in the service. The container must be joined to the le-backplane network for this connection to work.
 * $port - the port that the container is listening on. You can often see this by running 'docker ps' if not known.

```
server {

    listen 443 ssl;
    server_name $domain;

    ssl_certificate /etc/letsencrypt/live/$domain/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/$domain/privkey.pem;

    ssl on;
    ssl_session_cache  builtin:1000  shared:SSL:10m;
    ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers HIGH:!aNULL:!eNULL:!EXPORT:!CAMELLIA:!DES:!MD5:!PSK:!RC4;
    ssl_prefer_server_ciphers on;

    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;";

    access_log /var/log/nginx/access.log;

    # Can work around certain issues
    underscores_in_headers on;

    # To enable large uploads: remove these if desired
    client_max_body_size 10G;
    client_body_buffer_size 10m;

    # Use internal Docker DNS
    resolver 127.0.0.11;

    location / {

      proxy_set_header        Host $host;
      proxy_set_header        X-Real-IP $remote_addr;
      proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header        X-Forwarded-Proto $scheme;

      # Prevent nginx from exiting if upstream container is down
      set $upstream http://$container:$port;

      # Fix the ...It appears that your reverse proxy set up is broken" error.
      proxy_pass $upstream;

      # Mitigate httpoxy attack
      proxy_set_header Proxy "";

      proxy_set_header Referer http://$container:$port;
      proxy_set_header Origin http://$container:$port;

      proxy_pass_header Set-Cookie;

    }
}


```

## Troubleshooting

If something went wrong, then the le-certbot container will exit with a non-zero return code and more information should be available in the logs:
```
docker-compose logs
```

__NOTE:__ While troubleshooting, edit the docker-compose.yml file and put 'y' next to the LE_TEST environment variable. This will tell Certbot to request test certificates which won't count towards your limited allocation of genuine certificate requests. This means you can restart the environment as many times as you like without locking your letsencrypt account out for a period of time. See https://letsencrypt.org/docs/rate-limits/ for more information.

### Timeouts

You should be able to see the nginx webserver receiving the challenge requests from the letsencrypt acme challenge server in the docker-compose logs. If you can't see these, then incorrect DNS, port forwards, or a local firewall rules are the most likely culprits.

If you see timeout errors for the actual acme letsencrypt servers themselves, check the status page to see if they are down: https://letsencrypt.status.io/

Beyond this, much of the troubleshooting steps from the Certbot website apply: https://certbot.eff.org/docs/using.html

If you are still having trouble, I would recommend installing certbot natively on a test system and seeing if you can generate a single test certificate that way. Once you are certain that everything is working, trying checking your docker-compose.yml file over and bringing the environment up with LE_TEST set to 'y'.
