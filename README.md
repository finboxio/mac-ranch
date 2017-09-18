# Wrapper to deploy Rancher on Docker for Mac

This is basically an opinionated fork of the mac bootstrap script from [rancher/10acre-ranch](https://github.com/rancher/10acre-ranch) written by the Rancher team. Here's what I changed and why:

- Virtualbox instead of xhyve driver for docker-machine. I had issues with cross-host networking using xhyve (namely performance and IP predictability). In particular, I couldn't figure a working configuration with xhyve that allowed me to switch between running a local dev version of the rancher server via eclipse (outside docker), and a containerized release via Docker for Mac and have it be accessible to host VMs via the same IP address. YMMV

- External mysql support. By running rancher-server against an external mysql container I don't have to worry about losing my stacks when upgrading or trying out a development version of the server.

- Custom boot2docker image. At the time of this writing, cross-host networking doesn't work on the latest boot2docker image because of some [kernel support issues](https://forums.rancher.com/t/how-to-get-rancher-overlay-network-working-locally/3044/9). If this isn't the case anymore, let me know! The image is included in this repo but all credit goes to [this guy here.](http://www.odoko.co.uk/local-rancher-with-overlay-networking/)

- Support for specifying a different agent container version. The original script has an option for this but it looked like it was ignored.

- Cleaned up some other unused options (-r and -p)

### Prerequisites

- Docker for Mac
- docker-machine
- VirtualBox

### Usage

```
mac-ranch Usage:
    mac-ranch [opts]
    -c - Create world
    -d - Destroy world
    -h - Print this message
    -l - List hosts
    -b - boot2docker URL
    -M - Host memory in mb (default: 512)
    -n - Number of hosts (default: 3)
    -s - Server Container (default: rancher/server:latest)
    -u - Registry mirror URL (default: none)
    -m - External mysql url (format mysql://user:pass@host:port/db). The host can be a container name.

```

### Start mysql

```
    docker run -it -d -v rancher-mysql:/var/lib/mysql -e MYSQL_RANDOM_ROOT_PASSWORD=yes -e MYSQL_DATABASE=rancher -e MYSQL_USER=rancher -e MYSQL_PASSWORD=rancher -p 3336:3306 --name rancher-mysql mysql
```

### Deploy a cluster

```
mac-ranch [-m mysql://rancher-mysql] -c -n <number of hosts>
```

### Deploy a specific release

```
mac-ranch [-m mysql://rancher-mysql] -c -n <number of hosts> -s rancher/server:vX.Y.Z
```

### Add more hosts (server must already be running)

```
mac-ranch -n <number of hosts>
```

### List all the hosts

```
mac-ranch -l
```

### Destroy the cluster

```
mac-ranch -d
```

## Tips and Tricks

This already makes it incredibly easy to run a local rancher cluster for experiementation, development, etc. But to achieve even greater levels of rancher bliss, here are a few tips I use in my own environment:

#### Install the latest from github
```
    curl -s https://raw.githubusercontent.com/finboxio/mac-ranch/master/mac-ranch -o /usr/local/bin/mac-ranch && chmod +x /usr/local/bin/mac-ranch
```

#### [Set up a local registry mirror](https://github.com/docker/distribution/blob/master/docs/recipes/osx-setup-guide.md)
By default, every VM you're running the agent on will have to pull all of its images from a remote repository. If you have a locally running registry mirror, you can use the `-u` option to effectively share locally cached images across hosts. It makes a big difference, assuming you're mostly using images hosted in a public repostiory. I don't think it's currently possible to set this up against a private registry, but if you know otherwise please share.

> Note:
> Your localhost should be available to virtualbox on `10.0.2.2`. So if I have a local registry mirror running on port 45000, I'll want to use the option `-u http://10.0.2.2:45000`.

#### Add an alias
```
alias mr='mac-ranch -u http://10.0.2.2:45000 -m mysql://rancher-mysql'
```
I've got that in my bash profile, which lets me bring up a cluster using my local mysql and registry with a simple `mr -c`, add a host with `mr -n 1`, and tear it down with `mr -d`

#### Setup dns and reverse proxy containers

With my system configured to use 127.0.0.1 as a DNS server, running [finboxio/docker-dns](https://github.com/finboxio/docker-dns) via Docker for Mac alongside [jwilder/nginx-proxy](https://github.com/jwilder/nginx-proxy) lets me access this rancher server in my browser at [http://rancher.docker]().

Additionally running [finboxio/rancher-lb](https://github.com/finboxio/rancher-lb) as a global service (technically it only needs to be on dm-host-1) allows me to access my rancher services at `<service-name>.rancher`. Here's a basic configuration, running as a stack named `lb`:

lb/docker-compose.yml

```
haproxy:
  ports:
  - 80:80/tcp
  labels:
    io.rancher.scheduler.global: 'true'
    lb.haproxy.9090.frontend: 80/http
  image: finboxio/rancher-lb
```

lb/rancher-compose.yml

```
haproxy:
  health_check:
    port: 80
    interval: 2000
    initializing_timeout: 20000
    unhealthy_threshold: 3
    strategy: recreate
    response_timeout: 2000
    healthy_threshold: 2
  metadata:
    scope: service
    stats:
      port: 9090
    global:
    - maxconn 4096
    - debug
    domains:
    - http://rancher
```

With this configuration, you can expose any of your container ports as <service-name>.rancher just by adding the label `lb.haproxy.<port>.frontend=80/http` to your service (and adding a healthcheck to your service, as only healthy containers will be active). See the [rancher-lb](https://github.com/finboxio/rancher-lb) repo for a discussion of these and other available configuration options.

# License
Copyright (c) 2014-2016 [Rancher Labs, Inc.](http://rancher.com)

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

[http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
