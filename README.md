# ss-v2ray-docker

Shadowsocks-libev server with v2ray-plugin running in Docker.

[![Docker Build Status](https://img.shields.io/docker/cloud/build/mazy/ss-v2ray.svg)](https://hub.docker.com/r/mazy/ss-v2ray)

---

## Current version

+ [shadowsocks-libev](https://github.com/shadowsocks/shadowsocks-libev): 3.3.5
+ [v2ray-plugin](https://github.com/shadowsocks/v2ray-plugin): 1.3.1

## Usage

### Pull the image

I recommend use a specified tag instead of the "latest" tag. [This article explained why](https://medium.com/@mccode/the-misunderstood-docker-tag-latest-af3babfd6375).

```bash
docker pull mazy/ss-v2ray:v3.3.5-1.3.1
```

### Start the proxy in HTTP mode

+ Shell script

```bash
docker run \
    -d \
    --restart always \
    -p <server_address>:80:1080 \
    -e PASSWORD=<password> \
    mazy/ss-v2ray:v3.3.5-1.3.1
```

+ With docker-compose

```yaml
---
version: '3'

services:
  shadowsocks-obfs-docker:
    image: mazy/ss-v2ray:v3.3.5-1.3.1
    restart: always
    ports:
      - <server_address>:80:1080
    environment:
      PASSWORD: <password>
```

Then allow the port used by the shadowsocks in your firewall.

### Start the proxy in HTTPS or QUIC mode

I haven't tried this. You can refer to [Shadowsocks over websocket (HTTPS)](https://github.com/shadowsocks/v2ray-plugin#shadowsocks-over-websocket-https), [Shadowsocks over quic](https://github.com/shadowsocks/v2ray-plugin#shadowsocks-over-quic) and [Issue a cert for TLS and QUIC](https://github.com/shadowsocks/v2ray-plugin#issue-a-cert-for-tls-and-quic).

### Specifying additional arguments

You can also add additional arguments by specifying the environment variable `ARGS`, like `-e ARGS=<arguments>` in the script, or

```yaml
environment:
  ARGS: <arguments>
```

in `docker-compose.yaml`.

Usually, running with no additional arguments will be just fine.

For a full list of arguments, you can refer to [Shadowsocks libev - Usage](https://github.com/shadowsocks/shadowsocks-libev#usage) and `v2ray-plugin -h`.

## Case example

In my case, I'm running the shadowsocks-libev with v2ray-plugin, with Caddy as a websocket reverse proxy, behind a CloudFlare CDN.

### Shadowsocks and v2ray-plugin server

The shadowsocks and v2ray-plugin is configured running in HTTP mode and is listening to localhost:10001.

The `docker-compose.yml` is like this:

```yaml
---
version: '3'

services:
  ss-v2ray-docker:
    image: mazy/ss-v2ray:v3.3.5-1.3.1
    restart: always
    ports:
      - 127.0.0.1:10001:1080
    environment:
      PASSWORD: "a-really-secure-password"
```

### Caddy example

```Caddyfile
www.mysite.com {
  proxy /shadowsocks localhost:10001 {
    without /shadowsocks
    websocket
    header_upstream -Origin
  }
}
```

### Nginx example

```
location /shadowsocks {
  proxy_redirect off;
  proxy_pass http://127.0.0.1:10001;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  proxy_set_header Host $http_host;
}
```

### CloudFlare

I added an A record in CloudFlare DNS panel, which contains the domain, the IP address, with CDN enabled just like serving a normal website.

### Mac

```bash
brew cask install v2ray-plugin
brew cask install shadowsocksx-ng
```

Start shadowsocksx-ng application and add new server with these
properties:
|Config Name   |Value         |
|-------------:|:-------------|
|Address       |```www.mysite.com``` |
|Port          |*Port of your shadowsocks instance(80 or 443 if it behind webserver)*  |
|Encryption    |```chacha20-ietf-poly1305```|
|Password      |```password```|
|Plugin        |```path=/;mux=8;host=mazy.wtf;tls```|
|Plugin Opts   |```path=/shadowsocks;host=www.mysite.com;tls```|

### Linux client
*Tested on ArchLinux, should be similar on any distribution*

```
pacman -S shadowsocks-libev shadowsocks-v2ray-plugin
```

```/etc/shadowsocks/my-config.json```
```
{
	"server":"www.mysite.com",
	"server_port":443,
	"local_port":1080,
	"password":"password",
	"timeout":600,
	"method":"chacha20-ietf-poly1305",
	"plugin":"/usr/bin/v2ray-plugin",
	"fast_open":true,
	"plugin_opts":"path=/shadowsocks;host=www.mysite.com;tls",
	"reuse_port": true
}
```

```
systemctl enable --now shadowsocks-libev@my-config
```

I'm connecting to port 443 because I'm using the `Full SSL` configuration in CloudFlare, which means all HTTP requests will be rewrited into a HTTPS request. And for the same reason, I made the client running in the HTTPS mode, as in `plugin_opts` section you can see I specified the `tls` flag.

### Client on Android

First you need the Shadowsocks client and the v2ray plugin. You can install both of them from Google Play Store.

You can refer to the previous configuration or the config file above for the Shadowsocks part.

And for the plugin options, you can also refer to the config file above, but I think providing a little "translation" will be better.

|   Config Name|Value         |refer to `plugin_opts`    |
|-------------:|:-------------|:-------------------------|
|Transport mode|websocket-tls |tls                       |
|      Hostname|www.mysite.com|host=www.mysite.com       |
|          Path|/shadowsocks  |path=/shadowsocks         |

I left the `Concurrent connections` and `Certificate for TLS verification` untouched since I don't quite sure what does it means, and it just works.

For now you should have a working Shadowsocks-libev with v2ray-plugin proxy.
