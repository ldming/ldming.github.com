---
layout: post
title: "Ceph install document"
description: "Ceph install document"
category: Ceph
tags: [ceph, document]
---
{% include JB/setup %}

Ceph是一个基于对象的分布式文件存储系统，我们将其用于Totem对象代理数据库管理系统的底层数据存储。

Ceph集群包括四种类型的节点，client、Monitor、MDS和OSD，其中client对系统发出请求，Mon节点负责维护集群状态，包括monitor map、OSD map、pg map和crush map等，所有状态的变更都被记录；osd节点负责存储数据，数据复制，恢复，均衡，提供监控数据等；mds是可选组件，管理元数据，只有在使用ceph文件系统时才需要用到，负责userspace接口。

## 安装环境
 
* 操作系统：Ubuntu14.04 64位
* CPU：Intel 酷睿双核 1.8GHZ
* 硬盘：120GB


## 编译安装

1. 安装依赖包，如下：
 
    	apt-get install autotools-dev autoconf automake cdbs g++ gcc git libatomic-ops-dev libboost-dev libcrypto++-dev libcrypto++ libedit-dev libexpat1-dev libfcgi-dev libfuse-dev libgoogle-perftools-dev libgtkmm-2.4-dev libtool pkg-config uuid-dev libkeyutils-dev uuid-dev libkeyutils-dev  btrfs-tools libblkid-dev libudev-dev libsnappy-dev libleveldb-dev libboost-thread-dev  libboost-program-options-dev libaio-dev
2. `./autogen.sh`
3. `./configure --without-xfs`（此时如果还提示缺包，则安装指定的包，继续执行这一步，直到成功）
3. `sudo make`
4. `sudo make install`

## 目录结构

安装完成后，默认的安装目录在 `/usr/local/` 目录下，分别对应如下：

* `/usr/local/bin` 所有可执行文件包括需要用到的工具，共33个
* `/usr/local/etc/ceph/` 配置文件放置在目录下，包括两个配置文件 `ceph.conf` 和 `fetch_config`
* `/usr/local/share/doc/ceph/` 该目录下有两个配置文件样本，我们的配置文件需要从此处拷贝
* `/usr/local/lib` 存放所有的动态加载库文件
* `/var/lib/ceph/` 存放ceph相关组件的目录，包括Mon节点、OSD节点等的信息
* `/etc/ceph/ceph.conf` 在etc目录下还会存放一个ceph配置文件，其用途随后介绍

## 部署Mon节点

* 创建配置文件目录,与监控节点相关的信息保存在该目录下，需要在每个监控节点都创建该目录

	`sudo mkdir –p /var/lib/ceph/mon` 

* 使用uuidgen命令生成一个fsid，用来唯一标识一个cluster，如果产生的uuid为f649b128-963c-4802-ae17-5a76f36c4c76，则在ceph.conf中对应的配置参数如下：

	`fsid = f649b128-963c-4802-ae17-5a76f36c4c76	`

* 选择合适的集群名，例如集群名为ceph，则配置文件名为ceph.conf (这里集群名与配置文件名是对应的)

* 确定监控节点的名字

	`hostname –s` #节点名为ceph1

	IP为	`192.168.1.94` 

	则对应配置文件中的参数为：

	`mon initial members = ceph1`

	`mon host = ceph1`

	`mon addr = 192.168.1.94`

* 为集群创建一个keyring，并产生一个monitor密钥（secret key）

	`ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'`

* 产生一个管理员keyring，产生一个client.admin用户，并为该用户添加一个keyring
 
		ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --set-uid=0 --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow'

* 将client.admin的key添加至ceph.mon.keyring

	`ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring`

* 使用hostnames、host IP和fsid，产生一个monitor map，将其保存至/tmp/monmap

		monmaptool --create --add  ceph1 192.168.1.94  --fsid f649b128-963c-4802-ae17-5a76f36c4c76  /tmp/monmap 

	(如果有多个mon节点，则可以同时写多个节点的信息，类似`--add mon2 192.168.1.90 –add mon3 192.168.1.91`)

* 创建监控节点数据目录 

	`sudo mkdir /var/lib/ceph/mon/{cluster-name}-{hostname}`
 
	`sudo mkdir /var/lib/ceph/mon/ceph-ceph1`

* 使用前面创建的监控和管理key，以及map数据，写入数据目录

	`sudo ceph-mon --mkfs -i ceph1 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring`

* 最后的ceph.conf内容类似如下：

		[global]
		fsid = f649b128-963c-4802-ae17-5a76f36c4c76
	 	public network = 192.168.1.0/24
	 	cluster network = 192.168.1.0/24
		pid file = /var/run/ceph/$name.pid
		auth client required = cephx
		auth service required = cephx
		auth cluster required = cephx
 
		[mon]
		mon initial members = ceph1
		mon host = ceph1
		mon addr = 192.168.1.94
		mon data = /var/lib/ceph/mon/$cluster-$id

		[mon.ceph1]
		host = cehp1
		mon addr = 192.168.1.94

* 创建done文件，用于标识monitor已经创建并且准备好开始

	`sudo touch /var/lib/ceph/mon/ceph-ceph1/done`

*  启动该Mon节点

	`sudo /etc/init.d/ceph start ceph.ceph1`

* 如果要使该守护进程每次开机都启动，我们必须创建一个空的文件，如下：

	`sudo touch /var/lib/ceph/mon/ceph-ceph1/upstart`

### 注意：
在调用`/etc/init.d/ceph start mon.ceph1`启动`mon.ceph1`时会读取`/usr/local/etc/ceph/ceph.conf`这个配置文件，获取关于mon节点的一些信息，但是在使用`sudo ceph –s`这个命令时，同样也会读取配置文件，但此时读取配置文件却不是在`/usr/local/etc/ceph`这个目录下，而是在`/etc/ceph`这个目录下，因此我们需要将配置文件拷贝至该目录下一份，否则会报错，其提示如下：`Error initializing cluster client： Error`。

但是如果只有`/etc/ceph`下的配置文件，而没有`/usr/local/etc/ceph`下的配置文件，则在启动`mon.ceph1`时会提示该目录下没有找到对应的`ceph.conf`，个人猜想它们查找配置文件的目录应该是可以调整的。

**此时，如果配置成功的话，执行`ceph –s` 应该会输出类似以下的信息:**

	cluster f649b128-963c-4802-ae17-5a76f36c4c76
	     health HEALTH_ERR 64 pgs stuck inactive; 64 pgs stuck unclean; no osds
	     monmap e1: 1 ceph1 at {ceph1=192.168.1.94:6789/0}, election epoch 1, quorum 0 ceph1
	     osdmap e22: 0 osds: 0 up, 0 in
	      pgmap v69: 64 pgs, 1 pools, 0 bytes data, 0 objects
	            0 kB used, 0 kB / 0 kB avail
	                  64 creating


## 部署OSD节点

* 将key文件和ceph.conf拷贝至osd节点中，没有key则无法通讯

	`scp ceph.conf ceph.client.admin.keyring 192.168.1.95:/etc/ceph`

	此时，在每台机器上创建osd，需要使用一个uuid，执行以下命令uuidgen产生一个uuid

	`ceph osd create {uuid}` 产生一个osd-number 0,1,2等，表示该osd-number=0,1,2等

* 创建OSD数据目录
 
 	`mkdir /var/lib/ceph/osd/ceph-0`

* 创建文件系统，并将其挂在至相应的数据目录

	`sudo mkfs -t {fstype} /dev/{hdd}`

    `sudo mount -o user_xattr /dev/{hdd} /var/lib/ceph/osd/ceph-{osd-number}`

	 这里的`/dev/{hdd}`可以是磁盘分区，也可以是磁盘的逻辑分区（LVM）

* 初始化数据目录

	`sudo ceph-osd -i {osd-num} --mkfs --mkkey --osd-uuid [{uuid}] --cluster {cluster_name}`
  
	注意这里需要用到我们之前创建的uuid

* 注册osd认证key

	`ceph auth add osd.0 osd 'allow *' mon 'allow profile osd' -i /var/lib/ceph/osd/ceph-0/keyring`

* 添加ceph node至crush map

	`ceph osd crush add-bucket osd1 host` 这里的osd1为主机名

* 添加ceph node至default root

	`ceph osd crush move osd1 root=default` 这里的osd1为主机名

* 添加osd至crush map

	`ceph osd crush add osd.{osd_num}|{id-or-name} {weight} [{bucket-type}={bucket-name} ...]`

	`ceph osd crush add osd.0 1.0 host=osd1` 这里的osd1为主机名

* 启动osd服务

	`sudo /etc/init.d/ceph start osd.0`

* 添加服务 在每个	`/var/lib/ceph/osd/ceph-number` 目录下添加空白文件sysvinit

* 最后，通过`ceph –s` 查看集群状态

        cluster b24d2b17-28f2-419a-923d-e05332e2ea58
             health HEALTH_WARN 32 pgs degraded; 32 pgs stuck degraded; 64 pgs stuck unclean; 32 pgs stuck undersized; 32 pgs undersized; too few pgs per osd (8 < min 20)
             monmap e1: 1 mons at {ceph1=192.168.1.94:6789/0}, election epoch 2, quorum 0 ceph1
             osdmap e51: 8 osds: 8 up, 8 in
              pgmap v105: 64 pgs, 1 pools, 0 bytes data, 0 objects
                    8550 MB used, 140 GB / 156 GB avail
                          32 active+undersized+degraded
                          32 active+remapped
    
    **注意：**

    这里有一个警告: too few pgs per osd, 因此我们需要修改 `pg_num` , `pgp_num` 的数值如下:
    
    `ceph osd pool set rbd pg_num 266`
    
    `ceph osd pool set rbd pgp_num 266`
    
        cluster b24d2b17-28f2-419a-923d-e05332e2ea58
         health HEALTH_OK
         monmap e3: 1 mons at {ceph1=192.168.1.94:6789/0}, election epoch 1, quorum 0 ceph1
         mdsmap e2: 0/0/1 up
         osdmap e922: 8 osds: 8 up, 8 in
          pgmap v1788: 532 pgs, 2 pools, 0 bytes data, 0 objects
                8754 MB used, 139 GB / 156 GB avail
                     532 active+clean

    
* 通过 `ceph osd tree` 查看osd节点的状态

        # id	weight	type name	up/down	reweight
        -1	8	root default
        -2	4		host ceph1
        0	1			osd.0	up	1	
        1	1			osd.1	up	1	
        2	1			osd.2	up	1	
        3	1			osd.3	up	1	
        -3	4		host ceph2
        4	1			osd.4	up	1	
        5	1			osd.5	up	1	
        6	1			osd.6	up	1	
        7	1			osd.7	up	1	

## 部署MDS Server

* Edit `/usr/local/etc/ceph/ceph.conf` and add a MDS section like so:

        [mds]
            mds data = /var/lib/ceph/mds/$name
            keyring = /var/lib/ceph/mds/$name/keyring
        
        [mds.0]
            host = ceph1

* Create the authentication key(only if you use cephX)

    `mkdir /var/lib/ceph/mds/mds.0`
    
    `ceph-authtool -–create-keyring -–gen-key -n mds.0 /var/lib/ceph/mds/mds.0/keyring`
    
* Add the capabilities to that keyring. Also ran as root:

    `ceph auth add mds.0 osd ‘allow *’ mon ‘allow rwx’ mds ‘allow’ -i /var/lib/ceph/mds/mds.0/keyring`
    
* Start the service

    `/etc/init.d/ceph start mds.0`
    
* 如果我们修改了Mon所在主机的IP，通过以下方法修改Mon节点的信息

    1.修改ceph.conf中与Mon相关的IP信息
    2.`monmaptool --create --add ceph1 192.168.191.2 --fsid b24d2b17-28f2-419a-923d-e05332e2ea58 /tmp/monmap`
    3.`ceph-mon --inject-monmap /tmp/monmap --mon-data /var/lib/ceph/mon/ceph-ceph1/`
    
## Create a Ceph Filesystem

A Ceph filesystem requires at least two RADOS pools, one for data and one for metadata.

* Create data pool

    `ceph osd pool create cephfs_data 266`  # 266 is the pg_num
    
* Create metadata pool

    `ceph osd pool create cephfs_metadata 266` # 266 is the pg_num
    
* Create filesystem

    `ceph fs new <fs_name> <metadata> <data`
    
For example:
    
    `ceph fs new cephfs cephfs_metadata cephfs_data`
    
* Mount ceph fs

    `sudo mkdir /mnt/cephfs`
    
    `sudo mount -t ceph 192.168.1.1:6789:/ /mnt/cephfs`
    
* To mount the Ceph file system with `cephx` authentication enabled, you must specify a suer name and a secret.

    `sudo mount -t ceph 192.168.1.1:6789:/ /mnt/cephfs -o name=admin, secret=AQBOAKVUuKA3CxAAM6+hk7n/8rTJTi5gwoG20w==`
    
## 参考文献：

* 添加或移除mon节点，参考[这里](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-mons/)

* 添加或移除osd节点，参考[这里](http://docs.ceph.com/docs/master/rados/operations/add-or-rm-osds/)

* 手动部署Ceph手册：http://ceph.com/docs/master/install/manual-deployment/

* 德哥安装osd过程：http://blog.163.com/digoal@126/blog/static/163877040201411141846487/

* 德哥安装Mon过程：http://blog.163.com/digoal@126/blog/static/163877040201410274232276/

* 部署MDS过程：

    www.sebastien-han.fr/blog/2013/05/13/deploy-a-ceph-mds-server
    
    https://davespano.wordpress.com/2012/10/17/adding-an-mds-server-to-your-ceph-cluster/

* 整个部署过程以及遇到的问题：

    http://blog.zhaw.ch/icclab/deploy-ceph-and-start-using-it-end-to-end-tutorial-installation-part-13/

    http://blog.zhaw.ch/icclab/deploy-ceph-troubleshooting-part-23/
    
    http://blog.zhaw.ch/icclab/deploy-ceph-librados-client-part-33/
    
* Ceph Osd Reweight： http://cephnotes.ksperis.com/blog/2013/12/09/ceph-osd-reweight

* http://www.admin-magazine.com/HPC/Articles/Ceph-Maintenance

* 可能遇到的问题：

    http://ceph.com/docs/master/rados/troubleshooting/
    
    http://ceph.com/docs/next/cephfs/troubleshooting/