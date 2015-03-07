# 1
Run single application

Use Vagrant to run Docker containers

!SUB
Requirements

- Vagrant 1.7+
- Virtualbox 4.3+
- (optional) Docker 1.5 client


!SUB
Clone repository and start vagrant
```
git clone https://github.com/xebia/meetup-automating-the-modern-datacenter
cd meetup-automating-the-modern-datacenter
vagrant up --no-parallel
```

!SUB
(optional)
Set Docker environment variables:

```
$ export DOCKER_HOST=tcp://192.168.10.10:2375
$ unset DOCKER_TLS_VERIFY
$ unset DOCKER_CERT_PATH

$ docker ps
```

!SUB
Check application is running:
http://192.168.10.10

```
$ vagrant status

$ docker ps
```

!SLIDE
# 2
Application + database

Service registry + discovery using DNS


!SUB
No hardcoded links from app to dependencies

i.e. no:
```
redis = Redis(host='123.4.5.6', port=6379)
```


!SUB
Docker links are limited

- No cross host linking
- Can only link to 1 other container under a (service)name
- Can't update after the container is running


!SUB
Need service registry

## Consul :)


!SUB
Simplest way is using DNS

Add docker0 ip address as EXTRA_ARGS to Docker daemon & bind Consul as DNS server to the host


!SUB
Start Consul in a container
```
docker run -d -p 53:53/udp -p 8500:8500 cargonauts/consul-dns /consul agent -server -bootstrap-expect 1 -data-dir /tmp/consul -config-dir /opt/config/ -client 0.0.0.0
```


!SUB
Check that Consul is running
```
$ docker ps
CONTAINER ID        IMAGE                                 COMMAND                CREATED             STATUS              PORTS                                        NAMES
78f209ea00ba        cargonauts/consul-dns:latest          "/bin/sh -c '/consul   3 seconds ago       Up 1 seconds        0.0.0.0:53->53/udp, 0.0.0.0:8500->8500/tcp   condescending_payne
```

WebUI @ http://DOCKERHOSTIP:8500


!SUB
Manually add a service to Consul
```
curl -X POST http://consul.service.consul:8500/v1/agent/service/register \
  --header 'Content-Type: application/json' \
  --data-binary '{"ID": "helloworld1", "Name": "helloworld", "Address": "123.4.5.6", "Port": 80}'
```
& check if it's registered
```
docker run -ti cargonauts/toolbox
$ dig helloworld.service.consul +short
123.4.5.6
```

WebUI @ http://DOCKERHOSTIP:8500


!SUB
Remove the manually created service
```
curl -X POST \
	http://consul.service.consul:8500/v1/agent/service/deregister/helloworld1
```


!SUB
Automatically registering a service

Register from within the container after the process has started
```
$ docker run -d redis-with-wrapper
```
Check WebUI for new service


!SUB
Stopping the container doesn't unregister it
```
$ docker stop CONTAINERID
```
Check webUI, service is still there



!SUB
servicewrapper to register & unregister
```
$ docker stop
# works
$ docker kill
# doesn't
```


!SUB
Can't guarantee cleanup from within a container

So the service registry should be updated from outside the container


!SUB
Solution: listen to the Docker event stream

Using Jeff Lindsay's [registrator](https://github.com/gliderlabs/registrator)


!SUB
Automatically register & unregister a container using registrator



!SLIDE
# 3
Load-balancing the application

Service discovery using a proxy/loadbalancer

Using Consul-template



!SUB
Load-balancing can be done with Consul's DNS service

Round-robin

Limitations, e.g. caching


!SUB
Load-balancing using a proxy

Configured using Consule-template



!SLIDE
# 4
Health-checks

Update service registry


!SUB
Health-checks for the webserver apps

Remove from load-balancer if failed



!SUB
Dumb health-check
```
$ ping -c3 redis.service.consul
PING redis.service.consul (172.17.0.39): 48 data bytes
56 bytes from 172.17.0.39: icmp_seq=0 ttl=64 time=0.055 ms
56 bytes from 172.17.0.39: icmp_seq=1 ttl=64 time=0.083 ms
56 bytes from 172.17.0.39: icmp_seq=2 ttl=64 time=0.160 ms
```
(this only checks is the container is still running)


!SUB
Check of actual server is still running
```
$ nc -zv redis.service.consul 6379
Warning: inverse host lookup failed for 172.17.0.39: No address associated with name
redis.service.consul [172.17.0.39] 6379 (?) open
```


!SUB
Consul's included health-checks, e.g. HTTP


!SUB
Update the load-balancer based on the health of the apps

Using [Consul template](https://github.com/hashicorp/consul-template/) [with HAProxy](https://github.com/hashicorp/consul-template#examples)



!SLIDE
# 5
A/B testing


!SUB
Add new version of app to load-balancer
(helloworldp-python:1.1)



!SUB
Extra excercise: weighted load-balancing



!SLIDE
# 6/extra excercise
Feature toggles using Consul's K/V store