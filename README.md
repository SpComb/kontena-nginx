`nginx` loadbalancer for Kontena, using `confd` and `kontena-registrator`.

## Build

The `kontena/registrator` Docker image is not yet available on the Docker Hub.
You must first build the `kontena/registrator:dev` image locally:

    $ git clone github.com/SpComb/kontena-registrator
    kontena-registrator $ git submodule update --init
    kontena-registrator $ docker build -t kontena/registrator:dev

Then this stack can be built:

    $ kontena stack build --no-pull

This requires `--no-pull` to use the local `kontena/registrator:dev` image, instead of attempting to pull it from the Docker Hub.

## Install

    $ kontena stack install

## Issues

The `nginx` service container needs some proper pid-1 init to run both `nginx` and `confd`.
The current `init.sh` behaves badly if the daemonized nginx exits with an invalid nginx config, and the container will enter a Docker restart loop where confd is unable to fix the config.

This can happen if there is a service without any upstreams, and nginx rejects the config with an empty upstream..

## Usage

Configure your service with `io.kontena.nginx.*` envs:

```
services:
    whoami:
      environment:
        io.kontena.nginx.service: whoami
        io.kontena.nginx.tcp: 8000
        io.kontena.nginx.udp: 8000
        io.kontena.nginx.http: 8000
        io.kontena.nginx.http.location: /whoami
```

You do not need to link the service to the `nginx-lb` service.

### Load balancer

The `nginx-lb/nginx` service is configured using environment variables:

#### `LISTEN_IPV4=0.0.0.0`

Accept incoming traffic on the given IPv4 address.

#### `LISTEN_IPV6=::`

Accept incoming traffic on the given IPv6 address.

#### `LISTEN_HTTP=80`

Accept incoming HTTP connections on the given TCP port.

### Serivce containers

The `nginx-lb/registrator` registers Kontena service containers to etcd based on the container's `io.kontena.nginx.*` environment variables.

#### `io.kontena.nginx.service=SERVICE`

Register these Kontena service containers under the given `etcd:/kontena/nginx/*/SERVICE` path.

You may register multiple Kontena services under the same nginx-lb service, but they must have identical service configuration.

#### `io.kontena.nginx.udp=PORT`

Forward UDP traffic to the given port on the Kontena service container.

The load balancer accepts incoming UDP traffic using the same port.

#### `io.kontena.nginx.tcp=PORT`

Forward TCP connections to the given port on the Kontena service container.

The load balancer accepts incoming TCP connections using the same port.

#### `io.kontena.nginx.http=PORT`

Forward HTTP connections to the given port on the Kontena service container.

The load balancer accepts incoming HTTP connections on the `io.kontena.nginx.http.port` or `LISTEN_HTTP` port.

#### `io.kontena.nginx.http.location=/path`

Forward HTTP requests on the default vhost under the given URL prefix.

#### `io.kontena.nginx.http.host=HOST`

Forward HTTP requests on the given nginx `server_name`.

#### `io.kontena.nginx.http.aliases=ALIAS1 ALIAS2`

Forward HTTP requests on the given nginx `server_name` aliases.

#### `io.kontena.nginx.http.port=PORT`

Accept HTTP connections on the given port on the load balancer.

Defaults to the `LISTEN_HTTP` port.

## Schema

The `etcd` schema used.

#### `/kontena/nginx/http/:service/server`

```json
{
  "port": 8080,
  "host": "test.example.com",
  "aliases": "test",
  "location": "/test"
}
```

#### `/kontena/nginx/http/:service/upstreams/:upstream`

```json
{"ipv4": "10.81.128.0", "port": 8000}
```

#### `/kontena/nginx/tcp/:service/server`

```json
{"port": 8000}
```

#### `/kontena/nginx/tcp/:service/upstreams/:upstream`

```json
{"ipv4": "10.81.128.0", "port": 8000}
```

#### `/kontena/nginx/udp/:service/server`

```json
{"port": 8000}
```

#### `/kontena/nginx/udp/:service/upstreams/:upstream`

```json
{"ipv4": "10.81.128.0", "port": 8000}
```
