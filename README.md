# frp

[![Build Status](https://travis-ci.org/fatedier/frp.svg)](https://travis-ci.org/fatedier/frp)

[README](README.md) | [中文文档](README_zh.md)

## What is frp?

frp is a fast reverse proxy to help you expose a local server behind a NAT or firewall to the internet.Now, it supports tcp, http and https protocol when requests can be forwarded by domains to backward web services.

## Catalog

* [What can I do with frp?](#what-can-i-do-with-frp)
* [Status](#status)
* [Architecture](#architecture)
* [Example Usage](#example-usage) 
  * [Communicate with your computer in LAN by SSH](#communicate-with-your-computer-in-lan-by-ssh)
  * [Visit your web service in LAN by custom domains](#visit-your-web-service-in-lan-by-custom-domains)
* [Features](#features)
  * [Dashboard](#dashboard)
  * [Authentication](#authentication)
  * [Encryption and Compression](#encryption-and-compression)
  * [Reload configures without frps stopped](#reload-configures-without-frps-stopped)
  * [Privilege Mode](#privilege-mode)
    * [Port White List](#port-white-list)
  * [Connection Pool](#connection-pool)
  * [Rewriting the Host Header](#rewriting-the-host-header)
* [Development Plan](#development-plan)
* [Contributing](#contributing)
* [Contributors](#contributors)

## What can I do with frp?

* Expose any http and https service behind a NAT or firewall to the internet by a server with public IP address(Name-based Virtual Host Support).
* Expose any tcp service behind a NAT or firewall to the internet by a server with public IP address.
* Inspect all http requests/responses that are transmitted over the tunnel(future).

## Status

frp is under development and you can try it with latest release version.Master branch for releasing stable version when dev branch for developing.

**We may change any protocol and can't promise backward compatible.Please check the release log when upgrading.**

## Architecture

![architecture](/doc/pic/architecture.png)

## Example Usage

Firstly, download the latest programs from [Release](https://github.com/fatedier/frp/releases) page according to your os and arch.

Put **frps** and **frps.ini** to your server with public IP.

Put **frpc** and **frpc.ini** to your server in LAN.

### Communicate with your computer in LAN by SSH

1. Modify frps.ini, configure a reverse proxy named [ssh]:

  ```ini
  # frps.ini
  [common]
  bind_port = 7000

  [ssh]
  listen_port = 6000
  auth_token = 123
  ```

2. Start frps:

  `./frps -c ./frps.ini`

3. Modify frpc.ini, set remote frps's server IP as x.x.x.x:

  ```ini
  # frpc.ini
  [common]
  server_addr = x.x.x.x
  server_port = 7000
  auth_token = 123

  [ssh]
  local_port = 22
  ```

4. Start frpc:

  `./frpc -c ./frpc.ini`

5. Connect to server in LAN by ssh assuming that username is test:

  `ssh -oPort=6000 test@x.x.x.x`

### Visit your web service in LAN by custom domains

Sometimes we want to expose a local web service behind a NAT network to others for testing with your own domain name and unfortunately we can't resolve a domain name to a local ip.

Howerver, we can expose a http or https service using frp.

1. Modify frps.ini, configure a http reverse proxy named [web] and set http port as 8080, custom domain as `www.yourdomain.com`:

  ```ini
  # frps.ini
  [common]
  bind_port = 7000
  vhost_http_port = 8080

  [web]
  type = http
  custom_domains = www.yourdomain.com
  auth_token = 123
  ```

2. Start frps:

  `./frps -c ./frps.ini`

3. Modify frpc.ini and set remote frps server's IP as x.x.x.x. The `local_port` is the port of your web service:

  ```ini
  # frpc.ini
  [common]
  server_addr = x.x.x.x
  server_port = 7000
  auth_token = 123

  [web]
  type = http
  local_port = 80
  ```

4. Start frpc:

  `./frpc -c ./frpc.ini`

5. Resolve A record of `www.yourdomain.com` to IP `x.x.x.x` or CNAME record to your origin domain.

6. Now visit your local web service using url `http://www.yourdomain.com:8080`.

## Features

### Dashboard

Check frp's status and proxies's statistics information by Dashboard.

Configure a port for dashboard to enable this feature:

```ini
[common]
dashboard_port = 7500
```

Then visit `http://[server_addr]:7500` to see dashboard.

![dashboard](/doc/pic/dashboard.png)

### Authentication

`auth_token` in frps.ini is configured for each proxy and check for authentication when frpc login in.

Client that want's to register must set a global `auth_token` equals to frps.ini.

Note that time duration bewtween frpc and frps mustn't exceed 15 minutes because timestamp is used for authentication.

### Encryption and Compression

Defalut value is false, you could decide if the proxy will use encryption or compression whether the type is:

```ini
# frpc.ini
[ssh]
type = tcp
listen_port = 6000
auth_token = 123
use_encryption = true
use_gzip = true
```

### Reload configures without frps stopped

If your want to add a new reverse proxy and avoid restarting frps, you can use this function:

1. `dashboard_port` should be set in frps.ini:

  ```ini
  # frps.ini
  [common]
  bind_port = 7000
  dashboard_port = 7500
  ```

2. Start frps:

  `./frps -c ./frps.ini`

3. Modify frps.ini to add a new proxy [new_ssh]:

  ```ini
  # frps.ini
  [common]
  bind_port = 7000
  dashboard_port = 7500

  [new_ssh]
  listen_port = 6001
  auth_token = 123
  ```

4. Execute `reload` command:

  `./frps -c ./frps.ini --reload`

5. Start frpc and [new_ssh] is available now.

### Privilege Mode

Privilege mode is used for who don't want to do operations in frps everytime adding a new proxy.

All proxies's configurations are set in frpc.ini when privilege mode is enabled.

1. Enable privilege mode and set `privilege_token`.Client with the same `privilege_token` can create proxy automaticly:

  ```ini
  # frps.ini
  [common]
  bind_port = 7000
  privilege_mode = true
  privilege_token = 1234
  ```

2. Start frps:

  `./frps -c ./frps.ini`

3. Enable privilege mode for proxy [ssh]:

  ```ini
  # frpc.ini
  [common]
  server_addr = x.x.x.x
  server_port = 7000
  privilege_token = 1234

  [ssh]
  privilege_mode = true
  local_port = 22
  remote_port = 6000
  ```

4. Start frpc:

  `./frpc -c ./frpc.ini`

5. Connect to server in LAN by ssh assuming username is test:

  `ssh -oPort=6000 test@x.x.x.x`

#### Port White List

`privilege_allow_ports` in frps.ini is used for preventing abuse of ports in privilege mode:

```ini
# frps.ini
[common]
privilege_mode = true
privilege_token = 1234
privilege_allow_ports = 2000-3000,3001,3003,4000-50000
```

`privilege_allow_ports` consists of a specific port or a range of ports divided by `,`.

### Connection Pool

By default, frps send message to frpc for create a new connection to backward service when getting an user request.If a proxy's connection pool is enabled, there will be a specified number of connections pre-established.

This feature is fit for a large number of short connections.

1. Configure the limit of pool count each proxy can use in frps.ini:

  ```ini         
  # frps.ini
  [common]
  max_pool_count = 50
  ```

2. Enable and specify the number of connection pool:

  ```ini 
  # frpc.ini
  [ssh] 
  type = tcp
  local_port = 22
  pool_count = 10
  ```

### Rewriting the Host Header

When forwarding to a local port, frp does not modify the tunneled HTTP requests at all, they are copied to your server byte-for-byte as they are received. Some application servers use the Host header for determining which development site to display. For this reason, frp can rewrite your requests with a modified Host header. Use the `host_header_rewrite` switch to rewrite incoming HTTP requests.

```ini                                                             
# frpc.ini                                                         
[web] 
privilege_mode = true
type = http
local_port = 80
custom_domains = test.yourdomain.com
host_header_rewrite = dev.yourdomain.com
```

If `host_header_rewrite` is specified, the Host header will be rewritten to match the hostname portion of the forwarding address.

## Development Plan

* Support udp protocol.
* Support wildcard domain name.
* Url router.
* Load balance to different service in frpc.
* Debug mode for frpc, prestent proxy status in terminal.
* Inspect all http requests/responses that are transmitted over the tunnel.
* Frpc can directly be a webserver for static files.
* Full control mode, dynamically modify frpc's configure with dashboard in frps.
* P2p communicate by make udp hole to penetrate NAT.

## Contributing

Interested in getting involved? We would like to help you!

* Take a look at our [issues list](https://github.com/fatedier/frp/issues) and consider submitting a patch
* If you have some wanderful ideas, send email to fatedier@gmail.com.

**Note: We prefer you to give your advise in [issues](https://github.com/fatedier/frp/issues), so others with a same question can search it quickly and we don't need to answer them repeatly.**

## Contributors

* [fatedier](https://github.com/fatedier)
* [Hurricanezwf](https://github.com/Hurricanezwf)
* [vashstorm](https://github.com/vashstorm)
* [maodanp](https://github.com/maodanp)
* 
https://news.ycombinator.com/item?id=12273722
Comments from Hacker NEws


	Hacker News new | comments | show | ask | jobs | submit	login
Frp: A fast reverse proxy to help you expose a local server to the internet (github.com)
50 points by fatedier 2 hours ago | hide | past | web | 12 comments | favorite

 

add comment


	
tckr 1 hour ago [-]

I use https://ngrok.com/, it masks service ports and gives them unique domain names.
reply
	
CyberShadow 11 minutes ago [-]

Worth noting that ngrok is a non-free (trial-ware) service (all data is proxied through ngrok servers), whereas frp is self-hosted.
reply
	
hollander 30 minutes ago [-]

Is this something like ISA Server does? I've been looking for something like that without the price tag and this seems it!
reply
	
jon-wood 1 hour ago [-]

It also has an incredibly useful debug console for HTTP(S) which allows you to inspect requests as they come in, its absolutely priceless when working on webhooks.
reply
	
FooBarWidget 36 minutes ago [-]

Why is that useful? Doesn't your web app or web framework already log requests?
reply
	
captn3m0 29 minutes ago [-]

It logs the headers and request body along with the response for every single route. This means if the server sending you a webhook changes routes, or sends some extra content or does something unexpected, you have immediate visibility into it and can replay the request with a single click.
Pretty nifty.
reply
	
self_awareness 1 hour ago [-]

I use `ssh -R` to my VPS with public IP. It's possible to forward any protocol with this option, HTTP, RDP, etc.
reply
	
vincnetas 2 hours ago [-]

There is a nice service called weaved.com which allows you to ssh to a machine (in my case raspberry Pi) without public IP address. I think it also supports HTTP.
reply
	
adontz 1 hour ago [-]

Looks like most of the features can be covered by nginx+iptables, so it will be great if authors will state what can't be done at all or in some specific case.
reply
	
copperx 1 hour ago [-]

I'd love to hear how to use nginx+iptables to traverse a NAT.
reply
	
hollander 29 minutes ago [-]

It's probably the ease of use?!
reply
	
azzwacb9001 52 minutes ago [-]

Very useful.
reply




Guidelines | FAQ | Support | API | Security | Lists | Bookmarklet | DMCA | Apply to YC | Contact

Search:  

