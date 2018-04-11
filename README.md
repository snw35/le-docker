# le-docker
Letsencrypt docker compose.

This docker-compose file will automate the deployment of my letsencrypt images:

 * [le-certbot](https://github.com/snw35/le-certbot)
 * [le-nginx](https://github.com/snw35/le-nginx)

## How to use

 1. Install docker and docker-compose if needed.
 1. Make sure that port 80 and 443 are available on the machine/VM.
 1. Make sure that your domain resolves to the correct IP address.
 1. Make sure that port 80 and 443 are forwarded through your router/firewall to the machine, and that no firewall on the machine itself is blocking them.
 1. Create the two needed volumes and clone the repo:
```
docker volume create le-certs
docker volume create le-conf
git clone https://github.com/snw35/le-docker
cd le-docker
vi ./docker-compose.yml
```
 1. Enter your domain against LE_DOMAIN and your email address against LE_EMAIL, then bring the environment up:
```
docker-compose up -d
```

 1. The le-certbot container will attempt to generate a new certificate for the configured domain if none already exist. Wait about 30 seconds then check the logs to see if it succeeded:
```
docker-compose logs
```

## Where are my certs?

They are stored inside the le-certs volume. You can find them in:
```
/var/lib/docker/volumes/le-certs/_data/
```
Mounting this volume inside a container will give it access to your certificates, e.g:
```
docker run -d --mount source=le-certs,target=/etc/letsencrypt myimage:latest
```

## How do I configure nginx to proxy $app?

The nginx configuration is stored inside the le-conf volume. You can add new config files to it for each application / endpoint that you would like to serve/proxy. The safest way to do this is to use a temporary intermediate container to access the volume through docker like so:
```
docker run -dit --rm --mount source=le-conf,target=/etc/nginx/ alpine:3.7 sh
cd /etc/nginx
```
And then, when you are done, restart the compose environment to hup nginx:
```
docker-compose down
docker-compose up -d
```

Some good resources for creating nginx config files are:

 * Your application's official documentation ;)
 * Mozilla's SSL conf generator: https://mozilla.github.io/server-side-tls/ssl-config-generator/
 * Config generator by valentinxxx: https://nginxconfig.io/

## Troubleshooting

If everything went well, the le-certbot container will continue to run the crond daemon with 'certbot renew' running once per day, so your certificate will be automaticaly renewed once it gets close to its expiry date. If the container is stopped and run again, it will detect the existing certificates inside the le-certs volume and just go straight to running crond.

If something went wrong, then the container will exit with a non-zero return code and more information should be available in the logs:
```
docker-compose logs
```

While troubleshooting, edit the docker-compose.yml file and put 'y' next to the LE_TEST environment variable, then bring the environment down and back up again:

```
docker-compose down
docker-compose up -d
```

This will tell Certbot to request test certificates, which won't count towards your limited allocation of genuine certificate requests.

You should be able to see the nginx webserver receiving the challenge requests from the letsencrypt acme challenge server in the docker-compose logs. If you can't see these, then incorrect DNS, port forwards, or a local firewall rules are the most likely culprits.

Beyond this, much of the troubleshooting steps from the Certbot website will apply:

https://certbot.eff.org/docs/using.html


### Running on an Alpine host

__Note__ that certbot docker containers currently cannot run on an Alpine Linux host due to an [unresolved bug/issue](https://github.com/certbot/certbot/issues/5737). (This is Alpine as the main OS on which Docker is installed, not just as the base of the image).
