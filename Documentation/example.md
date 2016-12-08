Setup
=====

Prepare basic services
----------------------

```
# enable swarm mode
docker swarm init
# join all other nodes with supplied information

# create the cluster network
docker network create --driver=overlay overlay

# needed for CoreOS setup
ca_cert_dir=$(dirname $(readlink -ef /etc/ssl/certs/ca-certificates.crt ))

# start etcd cluster
docker pull steigr/coreos-etcd:v3.1.0-rc.1 && \
docker service create \
--replicas=3 --name=etcd \
--network=overlay \
--mount=type=bind,src=$ca_cert_dir,dst=/etc/ssl/certs \
--mount=type=volume,src=etcd-data,dst=/etcd \
--env=ETCD_INTERFACE=eth0 \
--env=ETCD_DISCOVERY=$(curl -sL 'https://discovery.etcd.io/new?size=3') \
steigr/coreos-etcd:v3.1.0-rc.1

#check if etcd cluster is up
etcd_member=$(docker ps --quiet --filter=label=com.docker.swarm.service.name=etcd | head -1)
docker exec $etcd_member etcdctl member list

# start torus cluster
docker pull steigr/coreos-torus:v0.1.2 && \
docker service create \
--replicas=5 --name=torus \
--network=overlay \
--mount=type=volume,src=torus-data,dst=/torus-data \
--env=TORUS_INTERFACE=eth0 \
--env=TORUSD_AUTO_JOIN=true \
--env=TORUSD_ETCD=etcd:2379 \
--env=TORUSD_SIZE=5GiB \
steigr/coreos-torus:v0.1.2

#check if torus cluster is up
torus_member=$(docker ps --quiet --filter=label=com.docker.swarm.service.name=torus | head -1)
docker exec $torus_member torusctl --etcd=etcd:2379 list-peers

# start torus-volume-driver
docker pull steigr/docker-volume-plugin-torus:v0.1.0 && \
docker service create \
--mode=global \
--mount=type=bind,source=/var/run,destination=/var/run \
--network=overlay \
--name=torusblk \
--env=TORUS_ETCD=etcd:2379 \
--env=DRIVER_NAME=torus \
steigr/docker-volume-plugin-torus:v0.1.0

docker pull minio/minio && \
docker service create \
--mount=type=volume,src=minio,dst=/minio,volume-driver=torus,volume-opt=size=5GiB \
--network=overlay \
--name=minio \
--publish=80:9000 \
minio/minio \
server /minio

df -h

minio_member=$(docker ps --filter=label=com.docker.swarm.service.name=minio --quiet | head -1)
docker logs $minio_member

# browse to http://$server-ip, login with credentials from `docker logs`, upload some data

# data has been copied into torus volume
df -h
```