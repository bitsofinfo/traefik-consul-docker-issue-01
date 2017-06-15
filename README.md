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

3. Ensure consul is up at http://localhost:8500, then make sure you are in this project's root and upload `example.toml` to consul

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

4. At this point the traefik config should be uploaded at: http://localhost:8500/ui/#/dc1/kv/traefik-stage/. Next Start traefik docker service

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

```
2017-06-15T18:00:37.559825827Z time="2017-06-15T18:00:37Z" level=debug msg="Configuration received from provider docker: {"backends":{"backend-nginx1":{"servers":{"server-nginx1-1":{"url":"http://10.0.4.7:80","weight":0}},"loadBalancer":{"method":"wrr"}}},"frontends":{"frontend-Host-mytest-mydomain1-com":{"entryPoints":["http"],"backend":"backend-nginx1","routes":{"route-frontend-Host-mytest-mydomain1-com":{"rule":"Host:mytest.mydomain1.com"}},"passHostHeader":true,"priority":0,"basicAuth":[]}}}"
2017-06-15T18:00:37.559845503Z time="2017-06-15T18:00:37Z" level=debug msg="Last docker config received more than 2s, OK"
2017-06-15T18:00:37.560149444Z time="2017-06-15T18:00:37Z" level=debug msg="Creating frontend frontend-Host-mytest-mydomain1-com"
2017-06-15T18:00:37.560187299Z time="2017-06-15T18:00:37Z" level=debug msg="Wiring frontend frontend-Host-mytest-mydomain1-com to entryPoint http"
2017-06-15T18:00:37.560207175Z time="2017-06-15T18:00:37Z" level=debug msg="Creating route route-frontend-Host-mytest-mydomain1-com Host:mytest.mydomain1.com"
2017-06-15T18:00:37.560226652Z time="2017-06-15T18:00:37Z" level=debug msg="Creating backend backend-nginx1"
2017-06-15T18:00:37.560460975Z time="2017-06-15T18:00:37Z" level=debug msg="Creating load-balancer wrr"
2017-06-15T18:00:37.560497731Z time="2017-06-15T18:00:37Z" level=debug msg="Creating server server-nginx1-1 at http://10.0.4.7:80 with weight 0"
2017-06-15T18:00:37.560517608Z time="2017-06-15T18:00:37Z" level=info msg="Server configuration reloaded on :80"
```

However if you go to the traefik dashboard at http://localhost:8080 nothing is displayed
