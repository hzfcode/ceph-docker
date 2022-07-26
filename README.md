
- [hosts](#hosts)
- [ceph docker images](#ceph-docker-images)
- [mon install](#mon-install)
  - [mon](#mon)
- [osd install](#osd-install)
  - [mon](#mon-1)
  - [osd](#osd)
- [mgr install](#mgr-install)
  - [mgr](#mgr)
- [rgw install](#rgw-install)
  - [mon](#mon-2)
  - [rgw](#rgw)
- [mds install](#mds-install)
  - [mds](#mds)
- [help](#help)
  - [mgr dashboard](#mgr-dashboard)
  - [mgr services](#mgr-services)
  - [ceph module](#ceph-module)
  - [cluster status](#cluster-status)
  - [pool manager](#pool-manager)
  - [mount rbd](#mount-rbd)
  - [snap](#snap)
  - [mount kvm](#mount-kvm)
  - [mount cephfs](#mount-cephfs)
  - [osd delete](#osd-delete)
  - [host delete](#host-delete)

## hosts

```shell
cat << EOF >> /etc/hosts
192.168.0.251 mon
192.168.0.251 osd0
192.168.0.10 osd1
192.168.0.11 osd2
EOF
```

## ceph docker images

```shell
docker pull ceph/daemon:master-7ef46af-nautilus-centos-7-x86_64
docker tag 6f68932df7dd ceph/daemon:latest
```

## mon install

### mon

```shell
docker run -d --net=host \
--name=mon \
-v /etc/localtime:/etc/localtime \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph:/var/lib/ceph \
-e MON_IP=192.168.0.251 \
-e CEPH_PUBLIC_NETWORK=192.168.0.0/24 \
ceph/daemon:latest mon
```

## osd install

### mon

```shell
cat << EOF >> /etc/ceph/ceph.conf
[osd.0]
bluestore_block_size=350G
[osd.1]
bluestore_block_size=800G
[osd.2]
bluestore_block_size=800G
EOF
scp /etc/ceph/ceph.conf root@osd:/etc/ceph/ceph.conf

docker exec mon ceph auth get client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring
scp /var/lib/ceph/bootstrap-osd/ceph.keyring root@osd:/var/lib/ceph/bootstrap-osd/ceph.keyring
```

### osd

```shell
docker run -d \
--name=osd \
--net=host \
--privileged=true \
-v /etc/localtime:/etc/localtime \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph:/var/lib/ceph \
-v /var/lib/ceph/osd:/var/lib/ceph/osd \
ceph/daemon:latest osd_directory
```

## mgr install

### mgr

```shell
docker run \
-d --net=host  \
--name=mgr \
-v /etc/localtime:/etc/localtime \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph:/var/lib/ceph \
ceph/daemon mgr
```

## rgw install

### mon

```shell
docker exec mon ceph auth get client.bootstrap-rgw -o /var/lib/ceph/bootstrap-rgw/ceph.keyring
```

### rgw

```shell
docker run \
-d --net=host \
--name=rgw \
-v /etc/localtime:/etc/localtime \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph:/var/lib/ceph \
ceph/daemon rgw
```

## mds install

### mds

```shell
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
```

## help

### mgr dashboard

```shell
docker exec mgr ceph mgr module enable dashboard
docker exec mgr ceph config set mgr mgr/dashboard/ssl false
docker exec mgr ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
docker exec mgr ceph config set mgr mgr/dashboard/server_port 7000
docker exec mgr ceph dashboard set-login-credentials admin 123456
```

### mgr services

```shell
docker exec mgr ceph mgr services
```

### ceph module

```shell
docker exec mgr ceph mgr module ls
```

### cluster status

```shell
ceph health detail
ceph -s
ceph -w
ceph quorum_status --format json-pretty
ceph osd df
ceph osd status
```

### pool manager

```shell
ceph osd pool ls
ceph osd pool create rbd.hzf 64
ceph osd pool ls detail
ceph tell mon.\* injectargs '--mon-allow-pool-delete=true'
ceph osd pool rm rbd.hzf rbd.hzf --yes-i-really-really-mean-it
```

### mount rbd

```shell
ceph osd pool create rbd.vm 64
ceph osd pool set rbd.vm size 1
ceph osd pool rm rbd.vm rbd.vm --yes-i-really-really-mean-it

ceph osd pool application enable rbd.vm rbd --yes-i-really-mean-it
rbd pool init rbd.vm

rbd create --size 50G rbd.vm/sg-216.img
rbd info rbd.vm/sg-216.img
rbd resize -s 2G rbd.vm/sg-216.img
rbd rm rbd.vm/sg-216.img

cat <<EOF > /etc/ceph/ceph.conf
[global]
mon host = 192.168.0.251
EOF

cat  <<EOF > /etc/ceph/ceph.client.admin.keyring
[client.admin]
        key = AQBpWNlihfYUIhAAd7PMmrfN8BvVlQevTUOhAg==
        caps mds = "allow *"
        caps mgr = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"
EOF

rbd map rbd.vm/sg-216.img
rbd feature disable rbd.vm/sg-216.img deep-flatten
rbd feature disable rbd.vm/sg-216.img fast-diff
rbd feature disable rbd.vm/sg-216.img object-map
rbd feature disable rbd.vm/sg-216.img exclusive-lock
fdisk -l
mkfs.xfs /dev/rbd0
mkdir /mnt/sg-216
mount /dev/rbd0 /mnt/sg-216
```

### snap

```bash
rbd snap create rbd.vm/sg-216.img@sg-216.img-osInit-20220726
rbd snap ls rbd.vm/sg-216.img
```

### mount kvm

```bash
ceph auth get-or-create client.libvirt mon 'profile rbd' osd 'profile rbd pool=rbd.vm'
qemu-img create -f rbd rbd.vm/sg-216 50G
```

### mount cephfs

```shell
ceph osd pool create cephfs/metadata 64
ceph osd pool create cephfs/data 64
ceph osd pool rm cephfs/metadata cephfs/metadata --yes-i-really-really-mean-it
ceph osd pool rm cephfs/data cephfs/data --yes-i-really-really-mean-it
ceph fs new cephfs cephfs/metadata cephfs/data
ceph fs ls
ceph fs rm cephfs --yes-i-really-mean-it
mount -t ceph 192.168.0.251:6789:/ /mnt/cephfs -o name=admin,secret=AQBNz85iBV5XKhAAJBvjfG1dNCVGhwLSlhADoA==
```

### osd delete

```bash
# 降osd权重
#   先降低osd权重为0，让数据自动迁移至其它osd，可避免out和crush remove操作时的两次水位平衡。
#   水位平衡完成后，即用ceph -s查看到恢复HEALTH_OK状态后，再继续后续操作。
# 注意
#   在生产环境下，若osd数据量较大，一次性对多个osd降权可能导致水位平衡幅度过大、云存储性能大幅降低，将影响正常使用。
#   应分阶段逐步降低osd权重，例如：从1.2降低至0.6，等待数据水位平衡后再降低至0。
ceph osd crush reweight osd.1 0
watch -n3 -d ceph -s
# 标记osd为out
ceph osd out osd.1
# 删除crush map中的osd
ceph osd crush remove osd.1
# 删除osd
ceph osd rm osd.1
# 删除auth
ceph auth del osd.1
```

### host delete

```bash
# 删除掉crush map中已没有osd的host。
ceph osd crush remove ubuntu
```
