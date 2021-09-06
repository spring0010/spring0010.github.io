[TOC]


# 背景

烽火台数据需要并入公司现有集群，需要增加现有集群的节点数，

公司集群使用CDH6.3.2安装，操作系统为Ubuntu 18.04（导致后续节点必须是Ubuntu 18.04，无法继续升级）

由于CDH现在已收费，官网已屏蔽所有非付费用户下载，想要扩展集群，需要使用寻找其他方式

**CDH无法解决的限制：**

* 操作系统只能是Ubuntu18.04（CDH官方还未支持20.04，18.04已是最新版）
* CDH内框架版本无法升级（hadoop，spark等）

# 可行方案

之前安装CDH6.3.2时，已下载过Cloudera Manager 6.3.1和CDH 6.3.2 parcels包，

我们已经拥有必需的包，但是无法直接安装，还缺少一些依赖包，这部分包可以选择手动安装（经测试与cloudera提供版本功能一致）

## 方案一：脚本安装

优点：

* 一键安装
* 脚本代码量少

缺点：

* 依赖本地安装文件（需要提前上传）
* 脚本鲁棒性不足，缺乏检测，启动失败概率较高（已遇到hostname不合法，hosts无法解析，权限不足等失败原因）
* 依赖外部源
* 强烈依赖49节点的本地仓库安装CDH组件
* 批量安装麻烦

具体流程：

1. 替换国内apt源

```shell
#替换国内源解决依赖下载不下来的问题
source_list='deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
'
echo "$source_list"> /etc/apt/sources.list.d/source.list
    apt apdate
```

2. 安装依赖和cloudera manager

```shell
#安装依赖
apt-get install -yq libsasl2-modules-gssapi-mit
apt-get install -yq libssl-dev
apt-get install -yq rpcbind
apt-get install -yq python-psycopg2
apt-get install -yq python-mysqldb
apt-get install -yq apache2
apt-get install -yq iproute2

#安装cloudera manager，须提前准备包
if [ -e cloudera-manager-daemons_6.3.1~1466458.ubuntu1804_all.deb ]&&[ -e cloudera-manager-agent_6.3.1~1466458.ubuntu1804_amd64.deb ];then
	dpkg -i cloudera-manager-daemons_6.3.1~1466458.ubuntu1804_all.deb
	dpkg -i cloudera-manager-agent_6.3.1~1466458.ubuntu1804_amd64.deb
fi
```

3. 启动cloudera manager

```shell
sed -i 's|server_host=localhost|server_host=bjuc49.optaim.com|' /etc/cloudera-scm-agent/config.ini
systemctl start cloudera-scm-agent
```

4.在webUI中安装CDH即可（49节点上有本地仓库）

## 方案二：搭建私有源

使用搭建私有源的方式，自建cloudera源，把cloudera-manager和CDH parcels上传，并解决证书问题（已解决，收费前下载的证书依然有效）

优点：

* 完全同收费前相同安装步骤
* 不依赖49本地仓库和本地安装包
* 批量安装可靠，速度快（cloudera内置BT下载）

缺点：需要提前搭建私有源

### 搭建私有源

1. 搭建一个静态http服务器（个人推荐使用nginx）

nginx.conf:

```nginx
#user nginx;
worker_processes 1;
error_log /data/cloudera_source/log/error.log;#错误日志存放位置
pid /data/cloudera_source/nginx.pid;#pid存放位置


events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /data/cloudera_source/log/access.log  main;#操作日志位置

    sendfile            on;#零拷贝
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;


    server {
        listen       10086;#服务端口
        #listen       [::]:10086;
        server_name  localhost;
        root         /usr/share/nginx/html;
        index index.html index.htm index.php;
    
    	#文件服务器配置
        location /cloudera {
            autoindex on;#自动增加索引
            autoindex_exact_size off;#关闭以字节为单位显示
            autoindex_localtime on;#显示时间
            root /data/cloudera_source;#文件目录
            }

        error_page 404 /404.html;
            location = /404.html {
            }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
            }
        }
}
```

2. 启动nginx

```shell
nginx -t -c /data/cloudera_source/nginx.conf
nginx -c /data/cloudera_source/nginx.conf 
```

3. 填充cloudera manager安装包

把之前下载的 cm6.3.1-ubuntu1804.tar.gz 解压至文件服务器的cm目录下

认证证书`allkeys.asc`也要拷贝至cm目录下

```shell
mkdir -p /data/cloudera_source/cm
tar zxvf cm6.3.1-ubuntu1804.tar.gz -C /data/cloudera_source/cm
cp allkeys.asc /data/cloudera_source/cm
```



4. 把CDH和FLINK的parcel包复制到文件服务器的cm目录下

parcel包和make_manifest.py先拷贝至本地

```shell
mkdir -p /data/cloudera_source/CDH/6.3.2
mv CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel /data/cloudera_source/CDH/6.3.2
sha1sum CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel |cut -d ' ' -f 1 > CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel.sha
cd /data/cloudera_source/CDH/6.3.2
make_manifest.py ./

mkdir -p /data/cloudera_source/CDH/GPLEXTRAS
mv GPLEXTRAS-6.3.2-1.gplextras6.3.2.p0.1605554-bionic.parcel /data/cloudera_source/CDH/GPLEXTRAS
sha1sum GPLEXTRAS-6.3.2-1.gplextras6.3.2.p0.1605554-bionic.parcel |cut -d ' ' -f 1 > GPLEXTRAS-6.3.2-1.gplextras6.3.2.p0.1605554-bionic.parcel.sha
cd /data/cloudera_source/CDH/GPLEXTRAS
make_manifest.py ./
```

```shell
mkdir -p /data/cloudera_source/FLINK/1.12.2
mv FLINK-1.12.2-csa1.0.0.0-cdh6.3.2-bionic.parcel /data/cloudera_source/FLINK/1.12.2
sha1sum FLINK-1.12.2-csa1.0.0.0-cdh6.3.2-bionic.parcel|cut -d ' ' -f 1 > FLINK-1.12.2-csa1.0.0.0-cdh6.3.2-bionic.parcel.sha
cd /data/cloudera_source/FLINK/1.12.2
make_manifest.py ./
```

5. 最终目录结构：

```shell
├── cloudera
│   ├── CDH
│   │   ├── 6.3.2
│   │   │   ├── CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel
│   │   │   ├── CDH-6.3.2-1.cdh6.3.2.p0.1605554-bionic.parcel.sha
│   │   │   └── manifest.json
│   │   └── GPLEXTRAS
│   │       ├── GPLEXTRAS-6.3.2-1.gplextras6.3.2.p0.1605554-bionic.parcel
│   │       ├── GPLEXTRAS-6.3.2-1.gplextras6.3.2.p0.1605554-bionic.parcel.sha
│   │       └── manifest.json
│   ├── cm
│   │   ├── allkeys.asc
│   │   ├── archive.key
│   │   ├── conf
│   │   │   └── distributions
│   │   ├── db
│   │   │   ├── checksums.db
│   │   │   ├── contents.cache.db
│   │   │   ├── packages.db
│   │   │   ├── references.db
│   │   │   ├── release.caches.db
│   │   │   └── version
│   │   ├── dists
│   │   │   ├── bionic-cm6
│   │   │   │   ├── contrib
│   │   │   │   │   ├── binary-amd64
│   │   │   │   │   │   ├── Packages
│   │   │   │   │   │   ├── Packages.gz
│   │   │   │   │   │   └── Release
│   │   │   │   │   └── source
│   │   │   │   │       ├── Release
│   │   │   │   │       └── Sources.gz
│   │   │   │   ├── InRelease
│   │   │   │   ├── Release
│   │   │   │   └── Release.gpg
│   │   │   ├── bionic-cm6.3
│   │   │   │   ├── contrib
│   │   │   │   │   ├── binary-amd64
│   │   │   │   │   │   ├── Packages
│   │   │   │   │   │   ├── Packages.gz
│   │   │   │   │   │   └── Release
│   │   │   │   │   └── source
│   │   │   │   │       ├── Release
│   │   │   │   │       └── Sources.gz
│   │   │   │   ├── InRelease
│   │   │   │   ├── Release
│   │   │   │   └── Release.gpg
│   │   │   └── bionic-cm6.3.1
│   │   │       ├── contrib
│   │   │       │   ├── binary-amd64
│   │   │       │   │   ├── Packages
│   │   │       │   │   ├── Packages.gz
│   │   │       │   │   └── Release
│   │   │       │   └── source
│   │   │       │       ├── Release
│   │   │       │       └── Sources.gz
│   │   │       ├── InRelease
│   │   │       ├── Release
│   │   │       └── Release.gpg
│   │   └── pool
│   │       └── contrib
│   │           ├── e
│   │           │   └── enterprise
│   │           │       ├── cloudera-manager-agent_6.3.1~1466458.ubuntu1804_amd64.deb
│   │           │       ├── cloudera-manager-daemons_6.3.1~1466458.ubuntu1804_all.deb
│   │           │       ├── cloudera-manager-server_6.3.1~1466458.ubuntu1804_all.deb
│   │           │       ├── cloudera-manager-server-db-2_6.3.1~1466458.ubuntu1804_all.deb
│   │           │       └── cloudera-manager-server-db_6.3.1~1466458.ubuntu1804_all.deb
│   │           └── o
│   │               └── oracle-j2sdk1.8
│   │                   └── oracle-j2sdk1.8_1.8.0+update181-1_amd64.deb
│   ├── FLINK
│   │   └── 1.12.2
│   │       ├── FLINK-1.12.2-csa1.0.0.0-cdh6.3.2-bionic.parcel
│   │       ├── FLINK-1.12.2-csa1.0.0.0-cdh6.3.2-bionic.parcel.sha
│   │       └── manifest.json
│   ├── jdk1.8.0_181-cloudera.tar
|   └── make_manifest.py
├── log
│   ├── access.log
│   └── error.log
├── nginx.conf
└── nginx.pid
```

### 安装

安装步骤同收费前，只需要注意，增加节点时使用自定义存储库，地址为上节建立的http文件服务器cm目录

如下图：

![image-20210906150814118](CDH%E6%89%A9%E5%AE%B9%E6%96%B9%E6%A1%88.assets/image-20210906150814118.png)

parcel配置远程存储库为自建库：

![image-20210906151048679](CDH%E6%89%A9%E5%AE%B9%E6%96%B9%E6%A1%88.assets/image-20210906151048679.png)

# 结论

通过搭建私有源可以最大限度满足目前基于CDH的扩展需求

