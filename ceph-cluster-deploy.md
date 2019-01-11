
## Ceph 部署

### 服务器节点准备

 hostname | role | os |
 -- | -- | -- | --|
 ceph-osd-n16 | OSD | CentOS 7.4 x86_64 |
 ceph-osd-n17 | OSD |  - |
 ceph-osd-n18 | OSD | - |
 ceph-mon-n14 | MON | - |
 ceph-mon-n21 | MON | - |
 ceph-mds-n19 | MDS | - |
 ceph-mds-n20 | MDS | - |

### 安装ceph-deploy
```bash
[root@ceph-mds-n19] yum install -y ceph-deploy
```
**如果出现部署失败，请手动下载最新版的**
 [ceph-deploy](https://download.ceph.com/rpm-kraken/el7/noarch/ "ceph-deploy")

**使用国内源站的话，则在环境变量申明以下变量**
```sh
export CEPH_DEPLOY_REPO_URL=http://mirrors.163.com/ceph/rpm-jewel/el7
export CEPH_DEPLOY_GPG_URL=http://mirrors.163.com/ceph/keys/release.asc
```
### 在每个节点安装ceph
```bash
#use ceph-deploy install {ceph-node1} {ceph-node2} ...
#ip host mapping in /etc/hosts
[root@ceph-mds-n19]  ceph-deploy install ceph-osd-n16 ceph-osd-n17 ceph-osd-n18 ceph-mon-n14 ceph-mon-n21 ceph-mds-n19 ceph-mds-n20
```

### 创建ceph集群并设置mon节点
```
#use <ceph-deploy new {ceph-node1} ... >to create a new ceph cluster and create ceph.conf in current directory
[root@ceph-mds-n19]  ceph-deploy  new ceph-mon-n14 ceph-mon-n21 ceph-mds-n19
#then you can edit ceph.conf before initial mon 
[root@ceph-mds-n19] vim ceph.conf 
#initial mon 
[root@ceph-mds-n19] ceph-deploy mon create-initial
```

### 添加osd节点
```
[root@ceph-mds-n19] ceph-deploy osd create ceph-osd-n16:/dev/vdb1 ceph-osd-n17:/dev/vdb1 ceph-osd-n18:/dev/vdb1
[root@ceph-mds-n19] ceph-deploy osd activate ceph-osd-n16:/dev/vdb1 ceph-osd-n17:/dev/vdb1 ceph-osd-n18:/dev/vdb1
```

### 创建 mds 节点
```
[root@ceph-mds-n19] ceph-deploy mds create ceph-mds-n19 ceph-mds-n20
```

### 创建cephfs
```
ceph osd pool create cephfs_data 64
ceph osd pool create cephfs_metadata 64
ceph fs new cephfs cephfs_metadata cephfs_data
```
### ceph-deploy

```
ceph-deploy new ceph-master ceph-node1 ceph-node2
ceph-deploy install ceph-master ceph-node1 ceph-node2
ceph-deploy mon create-initial
ceph-deploy osd prepare ceph-master:/dev/vdb2 ceph-node1:/dev/vdb2 ceph-node2:/dev/vdb2 
ceph-deploy osd activate ceph-master:/dev/vdb2 ceph-node1:/dev/vdb2 ceph-node2:/dev/vdb2
ceph-deploy admin ceph-master ceph-node1 ceph-node2
ceph-deploy admichmod +r /etc/ceph/ceph.client.admin.keyringn
ceph-deploy mds create ceph-node1 ceph-node2

ceph osd pool create cephfs_data 10
ceph osd pool create cephfs_metadata 10
ceph fs new cephfs cephfs_metadata cephfs_data

```
**如果出问题**

```
ceph-deploy purge ceph-master ceph-node1 ceph-node2
ceph-deploy purgedata ceph-master ceph-node1 ceph-node2
ceph-deploy forgetkeys
```

**客户端操作**
```sh
yum install ceph-common
mount -t ceph 10.2.0.38:6789:/ /d -o name=admin,secret=AQBd8rlaAHwEDRAAE1+tBB92FylzSPHEb3erGg==
```

### ceph.conf

```conf
[global]
fsid = b69ba88c-b53d-4f59-9284-5006c23948cb
mon_initial_members = ceph-master, ceph-node1, ceph-node2
mon_host = 10.154.9.122,10.154.19.13,10.133.196.252
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

##osd conf
osd pool default size = 3
osd pool default min size = 2
osd pool default pg num = 128
osd pool default pgp num = 128 
public network = 10.0.0.1/8 
cluster network = 10.0.0.1/8
max open files = 131072 


[mon]
mon data = /var/lib/ceph/mon/ceph-$id
mon clock drift allowed = 1                             #默认值0.05#monitor间的clock drift
mon osd min down reporters = 13                         #默认值1#向monitor报告down的最小OSD数
mon osd down out interval = 60  


[osd]
osd data = /var/lib/ceph/osd/ceph-$id
osd journal size = 20000 #默认5120                      #osd journal大小
osd journal = /var/lib/ceph/osd/$cluster-$id/journal #osd journal 位置
osd mkfs type = xfs                                     #格式化系统类型
osd mkfs options xfs = -f -i size=2048                  #强制格式化
filestore xattr use omap = true                         #默认false#为XATTRS使用object map，EXT4文件系统时使用，XFS或者btrfs也可以使用
filestore min sync interval = 10                        #默认0.1#从日志到数据盘最小同步间隔(seconds)
filestore max sync interval = 15                        #默认5#从日志到数据盘最大同步间隔(seconds)
filestore queue max ops = 25000                        #默认500#数据盘最大接受的操作数
filestore queue max bytes = 1048576000      #默认100   #数据盘一次操作最大字节数(bytes
filestore queue committing max ops = 50000 #默认500     #数据盘能够commit的操作数
filestore queue committing max bytes = 10485760000 #默认100 #数据盘能够commit的最大字节数(bytes)
filestore split multiple = 8 #默认值2                  #前一个子目录分裂成子目录中的文件的最大数量
filestore merge threshold = 40 #默认值10               #前一个子类目录中的文件合并到父类的最小数量
filestore fd cache size = 1024 #默认值128              #对象文件句柄缓存大小
journal max write bytes = 1073714824 #默认值1048560    #journal一次性写入的最大字节数(bytes)
journal max write entries = 10000 #默认值100         #journal一次性写入的最大记录数
journal queue max ops = 50000  #默认值50            #journal一次性最大在队列中的操作数
journal queue max bytes = 10485760000 #默认值33554432   #journal一次性最大在队列中的字节数(bytes)
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


[client]
rbd cache = true #默认值 true      #RBD缓存
rbd cache size = 335544320 #默认值33554432           #RBD缓存大小(bytes)
rbd cache max dirty = 134217728 #默认值25165824      #缓存为write-back时允许的最大dirty字节数(bytes)，如果为0，使用write-through
rbd cache max dirty age = 30 #默认值1                #在被刷新到存储盘前dirty数据存在缓存的时间(seconds)
rbd cache writethrough until flush = false #默认值true  #该选项是为了兼容linux-2.6.32之前的virtio驱动，避免因为不发送flush请求，数据不回写
rbd cache max dirty object = 2 #默认值0              #最大的Object对象数，默认为0，表示通过rbd cache size计算得到，librbd默认以4MB为单位对磁盘Image进行逻辑切分
rbd cache target dirty = 235544320 #默认值16777216    #开始执行回写过程的脏数据大小，不能超过 rbd_cache_max_dirty
```
