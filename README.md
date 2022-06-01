# Hosting multiple SSL-enabled sites with Docker and Nginx

Hosting multiple SSL-enabled sites with Docker and Nginx

Written by [Joel Hans](https://blog.ssdnodes.com/blog/author/joel/ "Joel Hans")

In one of our most popular tutorials—Host multiple websites on one VPS with Docker and Nginx—I covered how you can use the nginx-proxy Docker container to host multiple websites or web apps on a single VPS using different containers.

As I was looking to enable HTTPS on some of my self-hosted services recently, I thought it was about time to take that tutorial a step further and show you how to set it up to request Let's Encrypt HTTPS certificates.

With the help of docker-letsencrypt-nginx-proxy-companion (Github), we'll be able to have SSL automatically enabled on any new website or app we deploy with Docker containers
