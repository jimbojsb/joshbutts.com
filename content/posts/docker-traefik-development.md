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

### Step 2: Make a new project / Git repo

While it's certainly possible to apply these techniques to a single already existing project, the intention is that you're creating a new repo to store files and configuration for this system, which would sit above and outside any specific app that is connected to it. I like to use the top-level domain that I bought as the name of the project.
```bash
    [~/projects]$ mkdir joshbutts.dev
    [~/projects]$ cd joshbutts.dev
    [~/projects]$ git init
```
### Step 3: Set up the basics of Traefik

### Step 4: Set up Let's Encrypt for SSL

### Step 5: Connect an actual project to a subdomain

### Wrap Up