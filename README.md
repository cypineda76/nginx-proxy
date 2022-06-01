# nginx-proxy

Este proyecto es creado a partir del sitio (https://blog.ssdnodes.com/blog/host-multiple-ssl-websites-docker-nginx/)

# Hosting multiple SSL-enabled sites with Docker and Nginx

Written by [Joel Hans](https://blog.ssdnodes.com/blog/author/joel/ "Joel Hans")

In one of our most popular tutorials—[Host multiple websites on one VPS with Docker and Nginx](https://blog.ssdnodes.com/blog/host-multiple-websites-docker-nginx/)—I covered how you can use the `nginx-proxy` Docker container to host multiple websites or web apps on a single VPS using different containers.

As I was looking to enable HTTPS on some of my self-hosted services recently, I thought it was about time to take that tutorial a step further and show you how to set it up to request [Let's Encrypt](https://letsencrypt.org/) HTTPS certificates.

With the help of `docker-letsencrypt-nginx-proxy-companion` ([Github](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion)), we'll be able to have SSL automatically enabled on any new website or app we deploy with Docker containers.

## Prerequisites
- Any of our OS options—Ubuntu, Debian, or CentOS. Just a note: we've only tested Ubuntu 16.04 as of now.
- A running Docker installation, plus `docker-compose—see` [our Getting Started with Docker guide](https://blog.ssdnodes.com/blog/tutorial-getting-started-with-docker-on-your-vps/) for more information.

## Step 1. Getting set up, and a quick note
I'm a rather big fan of using `docker-compose` whenever possible, as it seems to simplify troubleshooting containers that are giving you trouble. Instead of parsing a long terminal command, you can simply edit the `docker-compose.yml` file, re-run `docker-compose up -d`, and see the results.

As a result, this tutorial will be heavily biased toward using `docker-compose` over `docker` commands, particularly when it comes to setting up the `docker-letsencrypt-nginx-proxy-companion` service.

If you're interested creating these containers via `docker` commands, check out the [docker-letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion#separate-containers-recommended-method) documentation.

*Already have `nginx-proxy` set up via our previous tutorial? You can simply overwrite your existing `docker-compose.yml` file with the file you'll find in Step 2*.

If you're just getting started with `nginx-proxy`, start here. You'll want to start by creating a separate directory to contain your `docker-compose.yml` file.

```
$ mkdir nginx-proxy && cd nginx-proxy
```
Once that's finished, issue the following command to create a unique network for `nginx-proxy` and other Docker containers to communicate through.
```
docker network create nginx-proxy
```

## Step 2. Creating the docker-compose.yml file
In the `nginx-proxy` directory, create a new file named `docker-compose.yml` and paste in the following text:
```
version: '3'

services:
  nginx:
    image: nginx:1.13.1
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - conf:/etc/nginx/conf.d
      - vhost:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - certs:/etc/nginx/certs
    labels:
      - "com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy=true"

  dockergen:
    image: jwilder/docker-gen:0.7.3
    container_name: nginx-proxy-gen
    depends_on:
      - nginx
    command: -notify-sighup nginx-proxy -watch -wait 5s:30s /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf
    volumes:
      - conf:/etc/nginx/conf.d
      - vhost:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - certs:/etc/nginx/certs
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - ./nginx.tmpl:/etc/docker-gen/templates/nginx.tmpl:ro

  letsencrypt:
    image: jrcs/letsencrypt-nginx-proxy-companion
    container_name: nginx-proxy-le
    depends_on:
      - nginx
      - dockergen
    environment:
      NGINX_PROXY_CONTAINER: nginx-proxy
      NGINX_DOCKER_GEN_CONTAINER: nginx-proxy-gen
    volumes:
      - conf:/etc/nginx/conf.d
      - vhost:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - certs:/etc/nginx/certs
      - /var/run/docker.sock:/var/run/docker.sock:ro

volumes:
  conf:
  vhost:
  html:
  certs:

# Do not forget to 'docker network create nginx-proxy' before launch, and to add '--network nginx-proxy' to proxied containers. 

networks:
  default:
    external:
      name: nginx-proxy
```
Special thanks to Github user [buchdag](https://github.com/buchdag) for this all-in-one configuration that works out of the box (most of the time).

After you copy, make sure that the formatting looks the same—the `yml` parser isn't kind, let's say, to syntax errors.

Now, take a closer look at line 30, which contains `- ./nginx.tmpl:/etc/docker-gen/templates/nginx.tmpl:ro`. This line creates a Docker volume between a file on your host filesystem (in this case, in the `nginx-proxy` directory) and a file inside one of the Docker containers. For that volume to work, we need to supply the `nginx.tmpl` file.

Inside of the `nginx-proxy` directory, use the following `curl` command to copy the developer's sample `nginx.tmpl` file to your VPS.
```
$ curl https://raw.githubusercontent.com/jwilder/nginx-proxy/master/nginx.tmpl > nginx.tmpl
```

## Step 3. Running nginx-proxy for the first time.
If you have an `nginx-proxy` container running already from the previous tutorial, you'll need to stop it before moving forward.

You can now run the `docker-compose` command that will kick this all off.
```
$ docker-compose up -d
```
The process will start by downloading a few Docker images, and if things finish successfully, the output will end with the following:
```
Creating nginx-proxy ...
Creating nginx-proxy ... done
Creating nginx-proxy-gen ...
Creating nginx-proxy-gen ... done
Creating nginx-proxy-le ...
Creating nginx-proxy-le ... done
```
To confirm this, run `docker ps`. You should see three running containers, named `nginx-proxy`, `nginx-proxy-gen`, and `nginx-proxy-le`, like this:
```
CONTAINER ID        IMAGE                                    COMMAND                  CREATED             STATUS              PORTS                                      NAMES
9ea5fffc24dd        jrcs/letsencrypt-nginx-proxy-companion   "/bin/bash /app/en..."   4 minutes ago       Up 4 minutes                                                   nginx-proxy-le
e2288dfc3c5c        jwilder/docker-gen:0.7.3                 "/usr/local/bin/do..."   4 minutes ago       Up 3 seconds                                                   nginx-proxy-gen
eda8f12bd829        nginx:1.13.1                             "nginx -g 'daemon ..."   4 minutes ago       Up 4 minutes        0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   nginx-proxy
```
If any of those aren't running, you should check their logs. You can do that with `docker logs`.
```
$ docker logs nginx-proxy
$ docker logs nginx-proxy-gen
$ docker logs nginx-proxy-le
```
I've found that most issues arise from `nginx-proxy-gen`. If you see an error about the `nginx-proxy` network, try creating the network again with `docker network create nginx-proxy`. If there are issues with the `nginx.tmpl` file, double check that it's the same as the [one on Github](https://raw.githubusercontent.com/jwilder/nginx-proxy/master/nginx.tmpl).

##

If your three containers are running smoothly, then you're ready to start deploying other SSL-enabled containers behind the proxy! At this point, we're shifting away from configuring `nginx-proxy` and toward the ways, you should configure your apps to run behind it.
