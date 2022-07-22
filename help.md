参考
    https://docs.ceph.com/
    https://quay.io/repository/ceph/daemon?tab=info
    http://www.chenlianfu.com/?p=3247
    https://blog.csdn.net/qq_27979109/article/details/120330224
    https://www.jianshu.com/p/cc3ece850433
    https://blog.csdn.net/hxx688/article/details/103440967

Ceph 社区最新版本是 17，而 Ceph 12 是市面用的最广的稳定版本。
拉取ceph docker镜像版本
curl -s -L https://quay.io/api/v1/repository/ceph/ceph/tag?page_size=100 | jq '."tags"[] .name'

x.0.z - 开发版（给早期测试者和勇士们）

x.1.z - 候选版（用于测试集群、高手们）

x.2.z - 稳定、修正版（给用户们）

| 版本名称 | 版本号 | 发布时间 |
| ------ | ------ | ------ |
| Argonaut | 0.48版本(LTS) | 　　2012年6月3日 |
| Bobtail | 0.56版本(LTS) | 　2013年5月7日 |
| Cuttlefish | 0.61版本 | 　2013年1月1日 |
| Dumpling | 0.67版本(LTS) | 　2013年8月14日 |
| Emperor | 0.72版本 | 　　　 2013年11月9 |
| Firefly | 0.80版本(LTS) | 　2014年5月 |
| Giant | Giant | 　October 2014 - April 2015 |
| Hammer | Hammer | 　April 2015 - November 2016|
| Infernalis | Infernalis | 　November 2015 - June 2016 |
| Jewel | 10.2.9 | 　2016年4月 |
| Kraken | 11.2.1 | 　2017年10月 |
| Luminous |12.2.12  | 　2017年10月 |
| mimic | 13.2.7 | 　2018年5月 |
| nautilus | 14.2.5 | 　2019年2月 |

## mimic新版本特性
- Bluestore
  * osd的新后端存储BlueStore已经稳定，是新创建的OSD的默认设置。
BlueStore通过直接管理物理HDD或SSD而不使用诸如XFS的中间文件系统，来管理每个OSD存储的数据，这提供了更大的性能和功能。
  * BlueStore支持Ceph存储的所有的完整的数据和元数据校验。
  * BlueStore内嵌支持使用zlib，snappy或LZ4进行压缩。（Ceph还支持zstd进行RGW压缩，但由于性能原因，不为BlueStore推荐使用zstd）
- 集群的总体可扩展性有所提高。我们已经成功测试了多达10,000个OSD的集群。
- mgr
  * mgr是一个新的后台进程，这是任何Ceph部署的必须部分。虽然当mgr停止时，IO可以继续，但是度量不会刷新，并且某些与度量相关的请求（例如，ceph df）可能会被阻止。我们建议您多部署mgr的几个实例来实现可靠性。
  * mgr守护进程daemon包括基于REST的API管理。注：API仍然是实验性质的，目前有一些限制，但未来会成为API管理的基础。
  * mgr还包括一个Prometheus插件。
  * mgr现在有一个Zabbix插件。使用zabbix_sender，它可以将集群故障事件发送到Zabbix Server主机。这样可以方便地监视Ceph群集的状态，并在发生故障时发送通知。

[global]#全局设置
fsid = xxxxxxxxxxxxxxx                           #集群标识ID 
mon host = 10.0.1.1,10.0.1.2,10.0.1.3            #monitor IP 地址
auth cluster required = cephx                    #集群认证
auth service required = cephx                           #服务认证
auth client required = cephx                            #客户端认证
osd pool default size = 3                             #最小副本数 默认是3
osd pool default min size = 1                           #PG 处于 degraded 状态不影响其 IO 能力,min_size是一个PG能接受IO的最小副本数
public network = 10.0.1.0/24                            #公共网络(monitorIP段) 
cluster network = 10.0.2.0/24                           #集群网络
max open files = 131072                                 #默认0#如果设置了该选项，Ceph会设置系统的max open fds
mon initial members = node1, node2, node3               #初始monitor (由创建monitor命令而定)
##############################################################
[mon]
mon data = /var/lib/ceph/mon/$id
mon clock drift allowed = 1                             #默认值0.05#monitor间的clock drift
mon osd min down reporters = 13                         #默认值1#向monitor报告down的最小OSD数
mon osd down out interval = 600      #默认值300      #标记一个OSD状态为down和out之前ceph等待的秒数
##############################################################
[osd]
osd data = /var/lib/ceph/osd/$id
osd mkfs type = xfs                                     #格式化系统类型
osd max write size = 512 #默认值90                   #OSD一次可写入的最大值(MB)
osd client message size cap = 2147483648 #默认值100    #客户端允许在内存中的最大数据(bytes)
osd deep scrub stride = 131072 #默认值524288         #在Deep Scrub时候允许读取的字节数(bytes)
osd op threads = 16 #默认值2                         #并发文件系统操作数
osd disk threads = 4 #默认值1                        #OSD密集型操作例如恢复和Scrubbing时的线程
osd map cache size = 1024 #默认值500                 #保留OSD Map的缓存(MB)
osd map cache bl size = 128 #默认值50                #OSD进程在内存中的OSD Map缓存(MB)
osd mount options xfs = "rw,noexec,nodev,noatime,nodiratime,nobarrier" #默认值rw,noatime,inode64  #Ceph OSD xfs Mount选项
osd recovery op priority = 2 #默认值10              #恢复操作优先级，取值1-63，值越高占用资源越高
osd recovery max active = 10 #默认值15              #同一时间内活跃的恢复请求数 
osd max backfills = 4  #默认值10                  #一个OSD允许的最大backfills数
osd min pg log entries = 30000 #默认值3000           #修建PGLog是保留的最大PGLog数
osd max pg log entries = 100000 #默认值10000         #修建PGLog是保留的最大PGLog数
osd mon heartbeat interval = 40 #默认值30            #OSD ping一个monitor的时间间隔（默认30s）
ms dispatch throttle bytes = 1048576000 #默认值 104857600 #等待派遣的最大消息数
objecter inflight ops = 819200 #默认值1024           #客户端流控，允许的最大未发送io请求数，超过阀值会堵塞应用io，为0表示不受限
osd op log threshold = 50 #默认值5                  #一次显示多少操作的log
osd crush chooseleaf type = 0 #默认值为1              #CRUSH规则用到chooseleaf时的bucket的类型
##############################################################
[client]
rbd cache = true #默认值 true      #RBD缓存
rbd cache size = 335544320 #默认值33554432           #RBD缓存大小(bytes)
rbd cache max dirty = 134217728 #默认值25165824      #缓存为write-back时允许的最大dirty字节数(bytes)，如果为0，使用write-through
rbd cache max dirty age = 30 #默认值1                #在被刷新到存储盘前dirty数据存在缓存的时间(seconds)
rbd cache writethrough until flush = false #默认值true  #该选项是为了兼容linux-2.6.32之前的virtio驱动，避免因为不发送flush请求，数据不回写
              #设置该参数后，librbd会以writethrough的方式执行io，直到收到第一个flush请求，才切换为writeback方式。
rbd cache max dirty object = 2 #默认值0              #最大的Object对象数，默认为0，表示通过rbd cache size计算得到，librbd默认以4MB为单位对磁盘Image进行逻辑切分
      #每个chunk对象抽象为一个Object；librbd中以Object为单位来管理缓存，增大该值可以提升性能
rbd cache target dirty = 235544320 #默认值16777216    #开始执行回写过程的脏数据大小，不能超过 rbd_cache_max_dirty

官方镜像
curl -s -L https://quay.io/api/v1/repository/ceph/ceph/tag?page_size=100 | jq '."tags"[] .name'
quay.io/ceph/daemon:latest-octopus

- 所有节点连接public网络
- osd节点连接私有网络
- 创建Ceph私有网络
docker network create --driver bridge --subnet 172.30.0.0/24 ceph

- 拉取搭建用镜像
docker pull ceph/daemon:latest-octopus

- 搭建mon节点
docker run -d --name mon --network ceph --ip 172.30.0.10 -p 3300:3300 -p 6789:6789 -e CLUSTER=ceph -e WEIGHT=1.0 -e MON_IP=172.30.0.10 -e MON_NAME=mon -e CEPH_PUBLIC_NETWORK=172.30.0.0/24 -v /etc/ceph:/etc/ceph -v /var/lib/ceph/:/var/lib/ceph/ -v /var/log/ceph/:/var/log/ceph/ ceph/daemon:latest-octopus mon

- 搭建osd节点
1 docker exec mon ceph auth get client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring

修改配置文件以兼容etx4硬盘
cat <<EOF >> /etc/ceph/ceph.conf
osd max object name len = 256
osd max object namespace len = 64
EOF

分别启动n个容器来模拟集群
sudo docker run -d --privileged=true --name osd-1 --network ceph --ip 172.30.0.11 -e CLUSTER=ceph -e WEIGHT=1.0 -e MON_NAME=ceph-mon -e MON_IP=172.30.0.10 -e OSD_TYPE=directory -v /etc/ceph:/etc/ceph -v /var/lib/ceph/:/var/lib/ceph/ -v /var/lib/ceph/osd/1:/var/lib/ceph/osd -v /etc/localtime:/etc/localtime:ro ceph/daemon:latest-octopus osd

```bash
for ((i=1;i<=3;i++));do
docker run -d --privileged=true --name osd-$i \
    --network ceph \
    --ip 172.30.0.1$i \
    -e CLUSTER=ceph \
    -e WEIGHT=1.0 \
    -e MON_NAME=mon \
    -e MON_IP=172.30.0.10 \
    -e OSD_TYPE=directory \
    -v /etc/ceph:/etc/ceph \
    -v /var/lib/ceph/:/var/lib/ceph/ \
    -v /var/lib/ceph/osd/$i:/var/lib/ceph/osd \
    -v /etc/localtime:/etc/localtime:ro \
    ceph/daemon:latest-octopus osd
done;
```

- 搭建mgr节点
docker run -d --privileged=true --name mgr --network ceph --ip 172.30.0.24 -e CLUSTER=ceph -p 7000:7000 --pid=container:mon -v /etc/ceph:/etc/ceph -v /var/lib/ceph/:/var/lib/ceph/ ceph/daemon:latest-octopus mgr

- 搭建rgw节点
docker exec mon ceph auth get client.bootstrap-rgw -o /var/lib/ceph/bootstrap-rgw/ceph.keyring
docker run -d --privileged=true --name rgw --network ceph --ip 172.30.0.25 -e CLUSTER=ceph -e RGW_NAME=rgw -p 7480:7480 -v /var/lib/ceph/:/var/lib/ceph/ -v /etc/ceph:/etc/ceph -v /etc/localtime:/etc/localtime:ro ceph/daemon:latest-octopus rgw

- 搭建mds节点
docker run -d --privileged=true --name mds --network ceph --ip 172.30.0.26 -e CLUSTER=ceph -e CEPHFS_CREATE=1 -v /var/lib/ceph/:/var/lib/ceph/ -v /etc/ceph:/etc/ceph ceph/daemon:latest-octopus mds

- 检查Ceph状态
docker exec mon ceph -s

- 测试添加rgw用户
docker exec rgw radosgw-admin user create --uid=admin --display-name=admin --access_key=admin --secret=tobeno.1

# CEPH分布式存储集群的全体关机和开机
由于Ceph集群的备份机制，若直接关闭某个主机，系统认为相应的OSDs挂掉，导致数据平衡操作发生，对数据的安全有一定影响。因此，关机需要对数据进行停止读写，并关闭备份机制，然后依次关闭纯OSD主机、元数据主机、监控主机、备用管理主机，最后关闭管理主机。开机的时候依次开启纯OSD主机、元数据主机、监控主机、管理主机和备用管理主机。

# 关闭数据读写和备份机制
ceph osd set noout
ceph osd set norecover
ceph osd set norebalance
ceph osd set nobackfill
ceph osd set nodown
ceph osd set pause

依次关闭纯OSD主机、元数据主机、监控主机、备用管理主机和管理主机
依次开启纯OSD主机、元数据主机、监控主机、管理主机和备用管理主机

# 开启数据读写和备份机制
ceph osd unset noout
ceph osd unset norecover
ceph osd unset norebalance
ceph osd unset nobackfill
ceph osd unset nodown
ceph osd unset pause

# 存储池基本管理命令
ceph osd pool    #OSD存储池管理命令
    ls：列出osd存储池列表；等同于(ceph osd lspools)
        detail：列出osd存储池列表详细信息
    application    #存储池关联应用程序管理；可关联应用程序[cephfs,rbd,rgw]
        get PoolName：获取存储池关联信息；不加存储池名为显示所有池关联信息
        enable PoolName ApplicationName：配置存储池关联应用程序
        disable PoolName ApplicationName：取消存储池关联应用程序
    create PoolName：创建osd存储池
    delete PoolName：删除osd存储池；必须在Monitor的配置中将mon_allow_pool_delete标志设置为true，否则将拒绝删除池；
    rename Srcpool Destpool：重命名osd存储池

# RBD基本管理命令
rbd    #Rados块设备(RBD)映像管理命令
    init: 初始化才能使用
    ls(list)：列出所有块设备列表
        PoolName：列出指定RBD池的块设备列表
    info PoolName/RBDName：显示块设备大小等信息；不指定RBD池默认显示默认池(rbd)的块设备信息；
    create    #块设备创建命令
        -s,–size Size    #定义块设备容量大小；默认单位为M；单位[M，G，T]
        PoolName/RBDName：定义rbd名称；不指定RBD池默认创建至默认池(rbd)中；
    resize -s Size PoolName/RBDName：调整块设备(扩大或缩小)的大小；不指定RBD池则为默认池(rbd)中的块设备；
    rm(remove) PoolName/RBDName：删除块设备；不指定RBD池则为删除默认池(rbd)中的块设备；
    map: 映射

# RBD features值
在创建镜像前我们还需要修改一下features值
在Centos7内核上，rbd很多特性都不兼容，目前3.0内核仅支持layering。所以我们需要删除其他特性
1：layering: 支持分层
2：striping: 支持条带化 v2
3：exclusive-lock: 支持独占锁
4：object-map: 支持对象映射(依赖 exclusive-lock)
5：fast-diff: 快速计算差异(依赖 object-map)
6：deep-flatten: 支持快照扁平化操作
7：journaling: 支持记录 IO 操作(依赖独占锁）
关闭不支持的特性一种是通过命令的方式修改，还有一种是在ceph.conf中添加rbd_default_features = 1来设置默认 features(数值仅是 layering 对应的 bit 码所对应的整数值)。
features编码如下
例如需要开启layering和striping，rbd_default_features = 3 (1+2)
属性               BIT码
layering             1
striping             2
exclusive-lock       4
object-map           8
fast-diff            16
deep-flatten         32

# 创建OSD磁盘
OSD服务是对象存储守护进程， 负责把对象存储到本地文件系统， 必须要有一块独立的磁盘作为存储。
如果没有独立磁盘，怎么办？ 可以在Linux下面创建一个虚拟磁盘进行挂载。
初始化10G的镜像文件：
mkdir  -p /usr/local/ceph-disk	
dd if=/dev/zero of=/usr/local/ceph-disk/ceph-disk-01 bs=1G count=10
将镜像文件虚拟成块设备：
losetup -f /usr/local/ceph-disk/ceph-disk-01
格式化：
#名称根据fdisk -l进行查询确认, 一般是/dev/loop0 
mkfs.xfs -f /dev/loop0 
挂载文件系统：
mkdir  -p /usr/local/ceph/data/osd/
mount /dev/loop0  /usr/local/ceph/data/osd/

# pool type
在请求大小越大的时候，erasure相对于replicated的写性能优势越发明显，当请求大小低于32KB时，erasure的写性能略低于replicated。
