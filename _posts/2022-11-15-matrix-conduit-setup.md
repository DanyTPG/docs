---
title: How to setup a conduit.rs Matrix server
date: 2022-11-15 12:00:00 +0330
categories: [Matrix, Conduit]
tags: [conduit,matrix]     # TAG names should always be lowercase
---


# Deploy using docker

This tutorial will be for deployments using docker.

We will need 2 containers. 
First a caddy webserver and the other a conduit.rs matrix server.

## configure caddy
For caddy we can use `caddy:alpine` official docker image. 

the Caddyfile to use:

```shell
matrix.example.com {
	reverse_proxy /_matrix/* <conduit-container-ip>:6167
	header /.well-known/matrix/* Content-Type application/json
	header /.well-known/matrix/* Access-Control-Allow-Origin *
	respond /.well-known/matrix/server `{"m.server": "matrix.example.com:443"}`
	respond /.well-known/matrix/client `{"m.homeserver":{"base_url":"https://matrix.example.com"},"m.identity_server":{"base_url":"https://identity.example.com"}}`
}
```
{: file='Caddyfile'}
the `"m.identity_server"` is not mandatory.

now run the container:

```shell
docker run -d --network host --restart unless-stopped \
-v /PATH/TO/Caddyfile:/etc/caddy/Caddyfile \
--name caddy caddy:alpine
```
## configure conduit
now for the conduit container.. first create a `conduit.toml` config file. sample could be found at the official conduit repo at <https://gitlab.com/famedly/conduit/>

Then we launch the container:

```shell
docker run -d --restart unless-stopped \
-v db:/var/lib/matrix-conduit/ \
-v /PATH/TO/conduit.toml:/etc/matrix-conduit/conduit.toml \
-e CONDUIT_CONFIG="/etc/matrix-conduit/conduit.toml" \
--name conduit matrixconduit/matrix-conduit:latest
```

All done!
