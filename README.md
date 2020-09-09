## GlusterFS快速安装

安装前准备
```shell script
# 这边最少需要三台机器,所有机器关闭防火墙和Selinux,所有机器需要两块硬盘
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i "s#SELINUX=enforcing#SELINUX=disabled#g" /etc/selinux/config

# 添加hosts文件,其实通过IP地址也能做集群,但是不建议这种方式,因为我们通过域名你就是替换节点ip地址只要是域名不变,我们的glusterfs集群还能使用
cat >> /etc/hosts<<EOF
192.168.12.25 glusterfs01
192.168.12.187 glusterfs02
192.168.12.15 glusterfs03
EOF

# 格式化硬盘,我们这边采用/dev/sdb 这个硬盘
fdisk /dev/sdb
Command (m for help): n
Partition type:
   p   primary (1 primary, 0 extended, 3 free)
   e   extended
Select (default p): p
然后一路回车,创建磁盘分区
mkfs.xfs /dev/sdb1
创建挂载点:
mkdir -p /guiji
mount -t auto /dev/sdb1 /data

# 服务端和客户端最好有个ntp时间服务器,确保机器时间一致,省略...
```

安装Glusterfs 服务
```shell script
# 添加安装源,如果不添加无法安装glusterfs-server
yum install -y centos-release-gluster 

# 安装glusterfs服务
yum install -y glusterfs glusterfs-server glusterfs-fuse glusterfs-rdma glusterfs-geo-replication glusterfs-devel

# 启动glusterfs服务
systemctl start glusterd
systemctl enable glusterd
```

## 初始化集群
```shell script
# 将节点加入到glusterfs集群中,需要到三台机器节点上都执行下面命令
gluster peer probe glusterfs01
gluster peer probe glusterfs02
gluster peer probe glusterfs03

# 查看gluster集群状态
[root@glusterfs01 ~]# gluster peer status
Number of Peers: 2

Hostname: glusterfs02
Uuid: d56bb3fa-46e0-4f81-8c2b-88f1c757ed1c
State: Peer in Cluster (Connected)

Hostname: glusterfs03
Uuid: 5c5c6b08-c0a4-419e-bb9e-0cfd85c5c17a
State: Peer in Cluster (Connected)

# 创建设置glusterfs volume,在三台机器上都创建如下目录
mkdir -p /data/pv1

# 然后在其中任何一台机器上执行下面这个命令
gluster volume create gfs replica 3 glusterfs01:/data/pv1 glusterfs02:/data/pv1 glusterfs03:/data/pv1
gluster volume start gfs 
# 注意: 这个地方默认是不可以直接使用挂载点开创建的,需要在挂载点下面创建子目录

#查看确认卷已经启动
[root@glusterfs01 ~]# gluster volume info
 
Volume Name: gfs                                   # 创建的这个glusterfs卷的名称,客户端挂在的时候回使用到这个内容
Type: Replicate                                    # 存储的类型,这个为副本类型
Volume ID: b5834223-5f0a-4514-a5da-742971dbff77
Status: Started                                    # 当前这个卷的状态
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3                        # 存储的状态
Transport-type: tcp
Bricks:
Brick1: glusterfs01:/data/pv1
Brick2: glusterfs02:/data/pv1
Brick3: glusterfs03:/data/pv1
Options Reconfigured:
transport.address-family: inet
storage.fips-mode-rchecksum: on
nfs.disable: on
performance.client-io-threads: off

# 查看状态
[root@glusterfs01 ~]# gluster volume status
Status of volume: gfs
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick glusterfs01:/data/pv1                 49152     0          Y       9847 
Brick glusterfs02:/data/pv1                 49152     0          Y       9846 
Brick glusterfs03:/data/pv1                 49152     0          Y       9832 
Self-heal Daemon on localhost               N/A       N/A        Y       9868 
Self-heal Daemon on glusterfs03             N/A       N/A        Y       9853 
Self-heal Daemon on glusterfs02             N/A       N/A        Y       9867 
 
Task Status of Volume gfs
------------------------------------------------------------------------------
There are no active volume tasks
```

## 客户端挂载写入数据
```shell script
# 安装软件,支持挂在glusterfs格式的文件
yum install centos-release-gluster -y
yum install -y glusterfs-6.0-1*   glusterfs-fuse-6.0-1*

# 在客户端也需要配置hosts解析 
cat >> /etc/hosts<<EOF
192.168.12.25 glusterfs01
192.168.12.187 glusterfs02
192.168.12.15 glusterfs03
EOF

# 挂载glusterfs提供的目录
mount -t glusterfs glusterfs01:/gfs /mnt
touch  /mnt/{1..10}
```

## GlusterFS的7种卷
```shell script
# 为了满足不同应用对高性能、高可用的需求，GlusterFS 支持 7 种卷，
# 基础卷：分布式卷（distribute）、条带卷（stripe）、复制卷（replica）、纠错卷（Dispersed ）
# 复合卷：分布式条带卷（distribute stripe）、分布式复制卷（distribute replica）、条带复制卷（stripe replica）、分布式条带复制卷（distribute stripe）

# 分布卷：存储数据时，将文件随机存储到各台glusterfs机器上。
#        优点：存储数据时，读取速度快
#        缺点：一个birck坏掉，文件就会丢失
# 复制卷：存储数据时，所有文件分别存储到每台glusterfs机器上。
#        优点：对文件进行的多次备份，一个brick坏掉，文件不会丢失，其他机器的brick上面有备份
#        缺点：占用资源
# 条带卷：存数据时，一个文件分开存到每台glusterfs机器上
#        优点：对大文件，读写速度快
#        缺点：一个brick坏掉，文件就会坏掉

# 1)分布式卷
[root@glusterfs01 ~]# mkdir -p /data/pv2/ /default_volume
[root@glusterfs01 ~]# gluster volume create default_volume glusterfs01:/data/pv2/ glusterfs02:/data/pv2/ glusterfs03:/data/pv2/
[root@glusterfs01 ~]# gluster volume start default_volume
[root@glusterfs01 ~]# mount -t glusterfs glusterfs02:/default_volume /default_volume
[root@glusterfs01 default_volume]# cd /default_volume/
[root@glusterfs01 default_volume]# touch {a..z}.txt
[root@glusterfs01 default_volume]# ls
a.txt  b.txt  c.txt  d.txt  e.txt  f.txt  g.txt  h.txt  i.txt  j.txt  k.txt  l.txt  m.txt  n.txt  o.txt  p.txt  q.txt  r.txt  s.txt  t.txt  u.txt  v.txt  w.txt  x.txt  y.txt  z.txt
[root@glusterfs01 ~]# ls /data/pv2/
b.txt  e.txt  l.txt  r.txt  x.txt  z.txt
[root@glusterfs02 ~]# ls /data/pv2/
a.txt  c.txt  d.txt  g.txt  k.txt  n.txt  o.txt  s.txt  u.txt  v.txt  w.txt  y.txt
[root@glusterfs03 ~]# ls /data/pv2/
f.txt  h.txt  i.txt  j.txt  m.txt  p.txt  q.txt  t.txt

# 2)复制式卷
[root@glusterfs01 ~]# mkdir /data/pv3
[root@glusterfs01 ~]# gluster volume create replica_volume replica 3 glusterfs01:/data/pv3/ glusterfs02:/data/pv3/ glusterfs03:/data/pv3/
[root@glusterfs01 ~]# gluster volume start replica_volume
[root@glusterfs01 ~]# mkdir /replica_volume
[root@glusterfs01 ~]# mount -t glusterfs glusterfs02:/replica_volume /replica_volume/
[root@glusterfs01 ~]# touch /replica_volume/{00..10}.txt
[root@glusterfs01 ~]# ls /data/pv3/
00.txt  01.txt  02.txt  03.txt  04.txt  05.txt  06.txt  07.txt  08.txt  09.txt  10.txt
[root@glusterfs02 ~]# ls /data/pv3/
00.txt  01.txt  02.txt  03.txt  04.txt  05.txt  06.txt  07.txt  08.txt  09.txt  10.txt
[root@glusterfs03 ~]# ls /data/pv3/
00.txt  01.txt  02.txt  03.txt  04.txt  05.txt  06.txt  07.txt  08.txt  09.txt  10.txt

# 3)条带卷
[root@glusterfs01 ~]# gluster volume create stripe-volume stripe 2 glusterfs01:/data/pv4/ glusterfs02:/data/pv4/ force
stripe option not supported

# 4)分布式条带卷
[root@node1 ~]# gluster volume create dis-stripe stripe 2 node1:/b3 node2:/b3 node3:/b3 node4:/b3 force
stripe option not supported

# 5)分布式复制卷
gluster volume create dis_rep_volume replica 2 glusterfs01:/data/pv5/ glusterfs02:/data/pv5/ glusterfs03:/data/pv5/ glusterfs04:/data/pv5/
gluster volume start dis_rep_volume
mkdir /dis_rep_volume
mount -t glusterfs glusterfs02:/dis_rep_volume /dis_rep_volume/
[root@glusterfs01 ~]# ls /data/pv5
00  01  02  03  04  05  08  a  b  c  e  g  j  k  l  m  n  o  p  q  r  t  u  w  y
[root@glusterfs02 ~]# ls /data/pv5
00  01  02  03  04  05  08  a  b  c  e  g  j  k  l  m  n  o  p  q  r  t  u  w  y
[root@glusterfs03 ~]# ls /data/pv5
06  07  09  10  d  f  h  i  s  v  x  z
[root@glusterfs04 ~]# ls /data/pv5
06  07  09  10  d  f  h  i  s  v  x  z
```
