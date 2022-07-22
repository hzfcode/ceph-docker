## hosts
cat << EOF >> /etc/hosts
192.168.0.251 mon
192.168.0.251 osd0
192.168.0.133 osd1
EOF

## ceph docker images
docker pull ceph/daemon:master-7ef46af-nautilus-centos-7-x86_64
docker tag 6f68932df7dd ceph/daemon:latest

## mon install

### mon

docker run -d --net=host \
--name=mon \
-v /etc/localtime:/etc/localtime \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph:/var/lib/ceph \
-e MON_IP=192.168.0.251 \
-e CEPH_PUBLIC_NETWORK=192.168.0.0/24 \
ceph/daemon:latest mon

## osd install

### mon
docker exec mon ceph auth get client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring

scp /etc/ceph/ceph.conf root@osd0:/etc/ceph/ceph.conf
scp /var/lib/ceph/bootstrap-osd/ceph.keyring root@osd0:/var/lib/ceph/bootstrap-osd/ceph.keyring

cat << EOF >> /etc/ceph/ceph.conf
[osd.0]
bluestore_block_size=350G
[osd.1]
bluestore_block_size=800G
[osd.2]
bluestore_block_size=800G
EOF

### osd

docker run -d \
--name=osd \
--net=host \
--privileged=true \
-v /etc/localtime:/etc/localtime \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph:/var/lib/ceph \
-v /var/lib/ceph/osd:/var/lib/ceph/osd \
ceph/daemon:latest osd_directory

## mgr install

### mgr

docker run \
-d --net=host  \
--name=mgr \
-v /etc/localtime:/etc/localtime \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph:/var/lib/ceph \
ceph/daemon mgr

## rgw install

### mon

docker exec mon ceph auth get client.bootstrap-rgw -o /var/lib/ceph/bootstrap-rgw/ceph.keyring

### rgw

docker run \
-d --net=host \
--name=rgw \
-v /etc/localtime:/etc/localtime \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph:/var/lib/ceph \
ceph/daemon rgw

## mds install

### mds

docker run -d \
--net=host \
--name=mds \
--privileged=true \
-v /etc/localtime:/etc/localtime \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph:/var/lib/ceph \
-e CEPHFS_CREATE=0 \
-e CEPHFS_METADATA_POOL_PG=512 \
-e CEPHFS_DATA_POOL_PG=512 \
ceph/daemon mds

## help

### mgr dashboard

docker exec mgr ceph mgr module enable dashboard
docker exec mgr ceph config set mgr mgr/dashboard/ssl false
docker exec mgr ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
docker exec mgr ceph config set mgr mgr/dashboard/server_port 7000
docker exec mgr ceph dashboard set-login-credentials admin 123456

### mgr services

docker exec mgr ceph mgr services

### ceph module

docker exec mgr ceph mgr module ls

### cluster status

ceph health detail
ceph -s
ceph -w
ceph quorum_status --format json-pretty
ceph osd df
ceph osd status

### pool manager

ceph osd pool ls
ceph osd pool create rbd/hzf 64
ceph osd pool ls detail
ceph tell mon.\* injectargs '--mon-allow-pool-delete=true'
ceph osd pool rm rbd/hzf rbd/hzf --yes-i-really-really-mean-it

### pool type

在请求大小越大的时候，erasure相对于replicated的写性能优势越发明显，当请求大小低于32KB时，erasure的写性能略低于replicated。

### mount rbd

yum install ceph-common
ceph osd pool create rbd 64
ceph osd pool set rbd size 1
ceph osd pool rm rbd rbd --yes-i-really-really-mean-it
ceph osd pool application enable rbd rbd --yes-i-really-mean-it
rbd pool init rbd
rbd create --size 1G rbd/rbd1.iso
rbd info rbd/rbd1.iso
rbd resize -s 2G rbd/rbd1.iso
rbd rm rbd/rbd1.iso
rbd map rbd/rbd1.iso
rbd feature disable rbd/rbd1.iso object-map
fdisk -l
mkfs.xfs /dev/rbd0
mkdir /mnt/rbd
mount /dev/rbd0 /mnt/rbd/

### mount cephfs

ceph osd pool create cephfs/metadata 64
ceph osd pool create cephfs/data 64
ceph osd pool rm cephfs/metadata cephfs/metadata --yes-i-really-really-mean-it
ceph osd pool rm cephfs/data cephfs/data --yes-i-really-really-mean-it
ceph fs new cephfs cephfs/metadata cephfs/data
ceph fs ls
ceph fs rm cephfs --yes-i-really-mean-it
mount -t ceph 192.168.0.251:6789:/ /mnt/cephfs -o name=admin,secret=AQBNz85iBV5XKhAAJBvjfG1dNCVGhwLSlhADoA==
