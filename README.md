# traefik-consul-docker-issue-01

https://github.com/containous/traefik/issues/1727

Attempt to have traefik gets its config from both consul and docker swarm, does not appear to work.

1. Create docker network

`docker network create -d overlay --attachable my-net`

2. Start consul locally

```
docker service create \
  --publish 8400:8400 \
  --publish 8500:8500 \
  --network my-net \
  --name consul \
  progrium/consul \
  -server \
  -bootstrap \
  -ui-dir /ui
```

3. Make sure you are in this project's root and upload `example.toml` to consul

```
export CDIR=`pwd`
docker run \
 -v $CDIR/example.toml:/etc/traefik/traefik.toml  \
 --network my-net \
 traefik \
 storeconfig \
 --consul \
 --consul.endpoint=consul:8500 \
 --consul.prefix="traefik-stage"
 ```

4. Start traefik docker service

 ```
 docker service create \
--name traefik \
--constraint=node.role==manager \
--mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock \
--network my-net \
 --publish 80:80 \
--publish 8080:8080 \
traefik  \
--consul \
--consul.endpoint=consul:8500 \
--consul.prefix=traefik-stage
```

5. Start nginx service

```
docker service create \
  --name nginx1 \
  --publish :80 \
  --label traefik.protocol=http \
  --label traefik.port=80 \
  --label traefik.frontend.rule='Host:mytest.mydomain1.com' \
  --label traefik.docker.network=my-net \
  --network my-net \
  nginx
```

At this point the traefik logs will properly show it creating the frontend/backends and routes for the nginx1 service that was detected from docker

However if you go to the traefik dashboard at http://localhost:8080 nothing is displayed
