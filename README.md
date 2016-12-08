This is a POC. USE WITH CAUTION!
================================

It is a torus volume driver for docker. It's swarmkit ready. Launch it like follows:

```
docker service create \
--mode=global \
--mount=type=bind,source=/var/run/docker/plugins/torus,destination=/var/run/docker/plugins/torus \
--mount=type=bind,source=/var/run/docker.sock,destination=/var/run/docker.sock \
--network=overlay \
--name=torusblk \
--env=TORUS_ETCD=etcd:2379 \
--env=DRIVER_NAME=torus \
steigr/docker-volume-plugin-torus
```

( Assuming you have etcd >= 3 and torus up an running, the container have to reach etcd under $TORUS_ETCD (can be vip or dnsrr or sth. as etcd-nodes should announce their correct unicast ip address, same goes for torusd).

Wait a moment until the service comes up (one per host). The container bootstraps a second container, as swarmkit does not allow privileged / host-pid-namespace containers.

`docker ps`

`docker volume ls`
`docker volume create --driver=torus --name=torus-demo --opt=size=5GiB --opt=fstype=xfs`
`docker volume ls`
`docker volume rm torus-demo`
`docker run --rm --volume=ghost-data:/var/lib/ghost --volume-driver=torus --publish=8080:2368 --name=ghost ghost`

Check log output of torusctl service.