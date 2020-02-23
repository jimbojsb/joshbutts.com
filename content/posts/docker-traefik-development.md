+++
date = 2020-02-22T00:00:00Z
slug = "traefik-docker-local-dev"
tags = ["docker", "traefik"]
title = "Using Traefik 2 and Docker for local development with multiple projects"

+++
### TL;DR

Fine, just skip directly to the Github repo that has a complete example of how to do this:  
[https://github.com/jimbojsb/docker-traefik-local-dev](http://github.com/jimbojsb/docker-traefik-local-dev)

### Background

### What is Traefik

Traefik is a reverse proxy.

### This seems more complicated than doing nothing...

### Step 1: Get a real domain name, and host it in public DNS

In order to get "real" SSL/TLS working, you need a real, publicly-registered-on-the-internet domain name. With the availability of `.dev` domains from Google's [get.dev](https://get.dev), my recommendation would be to buy one of these. There's also no reason you can't use a subdomain of an existing public domain name that you already own.

My perferred hosting provider for small personal projects is [Digital Ocean](https://www.digitalocean.com). For our purposes here, they are specifically nice because unlike AWS/Route53, they don't charge for hosting a domain in their DNS system. The rest of this article assumes you're using Digital Ocean's DNS, but other providers that are known to work with the Traefik certificate resolver are listed [here](https://docs.traefik.io/https/acme/#providers). If for some reason Digital Ocean doesn't work for you, I think [Cloudflare](https://cloudflare.com) would be my second choice.

The other upside of using a real public domain for this setup is that unless you need to work entirely offline (something that for me personally has become so rare that it's not worth worrying about), you don't need to run any sort of local DNS resolver like DNSMasq, as you might find recommended in various tutorial that are similar to this.

 Once you have a domain registered and properly set up with nameservers (again, in my case, this is going to be `ns1.digitalocean.com` and `ns2.digitalocean.com`), we need to create some `A` records. For my setup, I point *everything* to `127.0.0.1`, as I'm only going to be using this domain for a singular purpose. By pointing it all to `127.0.0.1`, any time we use that domain or a subdomain, it's expected to resolve right back to your local machine, where you'll be running Docker and Traefik.
 
Here's an example of what my setup in Digital Ocean looks like for `joshbutts.dev`:

![Digital Ocean local resolution DNS example](/images/posts/digitalocean-dns-local.png)

Notice the use of the wildcard to catch all subdomains. This means I never have to touch this again when I want to add more projects to this system locally. Note, most properly behaving DNS providers will resolve actually specified domains before wildcards, so if you want to have some other subdomains manually specified for some reason, you can still use a wildcard as the "catch all".

### Step 2: Make a new project / Git repo

While it's certainly possible to apply these techniques to a single already existing project, the intention is that you're creating a new repo to store files and configuration for this system, which would sit above and outside any specific app that is connected to it. I like to use the top-level domain that I bought as the name of the project.
```
[~/projects]$ mkdir joshbutts.dev
[~/projects]$ cd joshbutts.dev
[~/projects/joshbutts.dev]$ git init
[~/projects/joshbutts.dev (master)]$
```

### Step 3: Set up a new docker-compose.yml file

We're going to use Docker Compose in this new repo to spin up Traefik and an overlay network for our other apps to attach to. Below is the entire contents of the file, followed by explanations of what each part achieves.

```yaml
---
version: "3.7"
services:
  traefik:
    container_name: joshbutts.dev
    restart: always
    image: traefik
    networks:
      - joshbutts.dev
    ports:
      - 80:80
      - 443:443
      - 8080:8080
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./traefik:/etc/traefik:ro"
networks:
  joshbutts.dev:
    name: joshbutts.dev
```
As you can see, pretty short and simple but there's a decent amount of magic going on here that we need to discuss. We have just one service named `traefik` defined in this file. The name you choose for this service has no impact on anything.

###### image
```yaml
image: traefik
```
The Docker image we want to use comes off Dockerhub's public library of "blue ribbon" packages, so we can just reference it by the short name `traefik`. This also implies that you'll always be getting the latest version of the image. If that makes you nervous, you could specify someting more specific like `traefik:2.1`, which was the latest version as of this writing.

###### restart
```yaml
restart: always
```
This is up to your personal preference, but I like to add `restart: always` to this service because that way I don't have to remember to start or stop this when I go to start working on something. When idle Traefik consumes basically zero CPU and only a few MB of RAM.

###### container_name
```yaml
container_name: joshbutts.dev
```
Docker Compose will automatically name the containers under its control by combining the directory name of the project with the service name and a numeric suffix. This is fine, but also given that we know there will only ever be a single one of these, and I'm feeling particularly controlling, I'm overriding the name that will be generated for this container. Totally optional.

###### ports
```yaml
ports:
  - 80:80
  - 443:443
  - 8080:8080
```
This setup means Traefik will own ports `80`, `443` and `8080` on your local system. `80` and `443` will be used to host your HTTP applications, and for the purposes of this example, we're primarily concerned with 443 because we're going to set up SSL/TLS. `8080` is used for serve the Traefik API and Dashboard. `8080` is optional, but it's definitely useful especially when you're just getting started so you can see that your containers are all connected up correctly.

###### volumes
```yaml
volumes:
  - "/var/run/docker.sock:/var/run/docker.sock:ro"
  - "./traefik:/etc/traefik:ro"
```
We need to mount some volumes here in order for Traefik to be able to function correctly. The first is the Docker socket. Traefik will use this to listen to events on the Docker API so that it can automagically register new services when you start them via Docker. The second one is mounting a local directory we'll create in this project into the place where Traefik expects its configuration files to live by default. We mount both of these with the `:ro` flag because Traefik has no need to change or write to any of these files.

###### networks
```yaml
networks:
  joshbutts.dev:
    name: joshbutts.dev
```
Docker Compose will automatically (in recent versions) create an overlay network to connect your containers to. By default that network is named similarly to how containers are named, by using the directory and service names. Here we are overriding that naming, and then we also manually specify that overridden name in the service definition section as well. This override is optional but I strongly recommend you do it, because you'll need to specify this name in the external projects that you want to proxy with Traefik, and it's easier to remember a clean name (I'm using `joshbutts.dev` as the name so it matches domain name and the service name I specified) than it is to either go look up or try to remember how exactly Docker Compose named the network.
   
### Step 3: Set up the basics of Traefik

### Step 4: Set up Let's Encrypt for SSL

### Step 5: Connect an actual project to a subdomain

### Wrap Up