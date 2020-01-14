---
title: "Encrypting Your AWX Connection - Fun with Nginx"
date: 2018-01-10T19:30:10-05:00
draft: false
summary: "Putting Nginx in front of your AWX install will allow you to encrypt the connection."
tags: ["Docker", "Ansible", SSL, Nginx ]
---


**This article is out of date and may not reflect the best way to achieve this with newer versions of AWX. I have not used AWX in a long time so I will not be able to keep this article up to date with the newest information**

I've seen a few posts now in the [AWX Google Group](https://groups.google.com/forum/#!forum/awx-project) asking for help securing their AWX connection via SSL. There isn't currently a configurable way to do this with the default containers and installation procedure. I've heard that it's possible by modifying the nginx install within the awx_web container, but I don't like the idea of needing to do that with each new build. Instead, I created my own, minimally configured nginx container to serve as an SSL termination point. I decided to write this up as a (hopefully) short guide to help anyone else that might want to add containerized SSL termination in front of a web app, [AWX](https://github.com/ansible/awx) being my example.

**NOTE**: any reverse proxy would do the trick and haproxy is another good option (maybe even better), but since I've had a bit more experience with nginx, it's what I went with.

# Setting up Docker Build Workspace

I will make the assumption that you have a local Docker install and test this on, so first lets create a directory structure to work with for building the container. Don't worry if you've never built a container before, it's pretty easy and I'll try to hit every step necessary.

Another assumption I'm going to make is that you already have either a wildcard SSL certificate for your domain or a site cert for your AWX install.

If you have a certificate chain from your CA, you will need to combine it with your certificate.

*Combining cert with chain:*

```bash
cat example.com.crt bundle.crt > example.com.chained.crt
```

If the above was necessary for you, you will be putting the chained version of your certificate in the ~/my_nginx_build directory along with the ssl key. We will be assuming a chained cert name in the rest of the instructions.

```bash
mkdir ~/my_nginx_build && cd ~/my_nginx_build
touch Dockerfile nginx.conf
cp /path/to/ssl_cert/example.com.chained.crt .
cp /path/to/ssl_key/example.com.key .
```

# Creating the Docker and Nginx Configs

We will be utilizing the official nginx container built with alpine linux from Dockerhub. Below is what we will populate the Dockerfile with in our workspace. Be sure to replace the names of the example ssl certificate and key with the ones you're using.

*~/my_nginx_build/Dockerfile*:

```docker
FROM nginx:alpine
RUN mkdir -p /etc/ssl
COPY example.com.chained.crt /etc/ssl/example.com.chained.crt
COPY example.com.key /etc/ssl/example.com.key
COPY nginx.conf /etc/nginx/nginx.conf
```

And now for the nginx config file that we referenced in the Dockerfile.

*~/my_nginx_build/nginx.conf*:

```nginx

user nginx;
worker_processes 1;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    server {
        listen      80;
        server_name awx.example.com;
        rewrite     ^  https://$host$request_uri? permanent;
    }
    server {
        listen              443;
        server_name         awx.example.com;
        ssl                 on;
        ssl_certificate     /etc/ssl/example.com.chained.crt;
        ssl_certificate_key /etc/ssl/example.com.key;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         HIGH:!aNULL:!MD5;
        location / {
            proxy_set_header    Host $host;
            proxy_pass          http://awx_web:8052;
            proxy_http_version  1.1;
            proxy_set_header    Upgrade $http_upgrade;
            proxy_set_header      Connection "upgrade";
        }
    }
}
```

(NOTE: I edited this page after the original post and removed line numbers from my syntax highlighter. This is because of copy/paste issues this introduced. Sorry for the inconvenience.)

Lets unpack a couple things from the nginx config.

1. We are going to listen on both 80 and 443 but force any plain http going to port 80 to redirect to the https equivalent.
2. Lines 28-30 are important. This is what allows the web socket connections to work properly. This is mostly noticable when you are running a job in awx and watching the output of the job. [WebSocket](https://en.wikipedia.org/wiki/WebSocket) is utilized in AWX to allow the logs to continually be printed out without refreshing the page. While WebSockets get used all over the place in AWX, the log output being broken was the first thing I noticed when WebSockets was NOT functioning properly.
3. You'll notice on line 27 I use `awx_web` as the url. This is because once we launch this container it will need to be linked to the `awx_web` container. When linked, `awx_web` will be added to the `/etc/hosts/` file of this container along with the internal IP address of its container. Also, port `8052` is the default port used by the awx_web container.
4. The lines above the `http` directive I just pulled from the default nginx.conf

# Build and run nginx container

Ok now it's time for the magic. Lets build the container!

```bash
docker build -t my-nginx .
```

Your `docker image ls` command should return something along these lines:

```bash
#docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
my-nginx            latest              09d0af326ae8        5 seconds ago       16.8MB
nginx               alpine              bb00c21b4edf        20 hours ago        16.8MB
```

For this post, we'll just assume you're going to launch this next to your awx containers locally.

```bash
docker run -d --name my-nginx -p 80:80 -p 443:443 --link awx_web:awx_web my-nginx
```

One last thing to ensure nginx thinks you're navigating to your actual site, in this case: `awx.example.com`

```bash
sudo echo "127.0.0.1  awx.example.com" >> /etc/hosts
```

Now you should be ready to test! Please let me know on [twitter](https://twitter.com/localhost_luke) or the comments below if you have issues or suggestions on how to improve these instructions.


**Additional Info (Added Feb. 6, 2018)**:

Thanks to Florian in the comments below for pointing this out to me!

The above steps are assuming we're deploying WITHOUT using the default [installation steps](https://github.com/ansible/awx/blob/devel/INSTALL.md) provided by the AWX team. I should have mentioned this in the original post.
The main hangup with the default installation steps is that they will set up a port forward for `localhost:{host_port} => awx_web:8052`, where `host_port` is set by the inventory file they provide you. I would argue that you wouldn't want this forward in place at all if you're using nginx to handle all the traffic. As pointed out by Florian below, the easiest way to work around this is to just set the `{host_port}` var in the inventory to something we don't use (port 8052 will still be exposed in the docker network). Once moving this out of a testing phase, you most likely won't be using the "local_docker" role in the AWX installer steps, so it would be a non-issue. 