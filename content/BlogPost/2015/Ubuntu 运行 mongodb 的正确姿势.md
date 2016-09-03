曾经在多个项目中使用 mongodb，但是最近从头开始安装并启动居然遇到了不少问题。在此记录一下：

环境：Ubuntu 12.04 64bit  
版本：mongodb 3.0.2

首先，按照[官方教程][1]安装 mongodb。在这个教程中，注意到有一句话

> The MongoDB instance stores its data files in /var/lib/mongodb and its log files in /var/log/mongodb by default, and runs using the mongodb user account. You can specify alternate log and data file directories in /etc/mongod.conf.

恩，看来意思是我们可以通过改 `/etc/mongod.conf` 这个配置文件来设置 data 存放路径。
打开这个文件看看，的确是有这些配置。

```
# Note: if you run mongodb as a non-root user (recommended) you may
# need to create and set permissions for this directory manually,
# e.g., if the parent directory isn't mutable by the mongodb user.
dbpath=/var/lib/mongodb

#where to log
logpath=/var/log/mongodb/mongod.log
```

这里假设要用以下命令（称为命令1）启动 mongodb ，指定 wiredTiger 作为存储引擎并标示副本集:

	mongod --storageEngine wiredTiger --replSet fbt   # 称为命令1

首先我们非常天真地直接执行上面那行命令：

	2015-04-23T14:55:16.152+0800 I STORAGE  [initandlisten] exception in initAndListen: 29 Data directory /data/db not found., terminating
	2015-04-23T14:55:16.152+0800 I CONTROL  [initandlisten] dbexit:  rc: 100

Oops，似乎不行，说 `/data/db` 不存在。这里就是很坑爹的地方了：**虽然 `mongod.conf` 指定了 `dbpath`，但是实际上这个配置文件没有起作用！而 mongodb 本身认为 `/data/db` 才是默认路径。**毫无疑问，文档在这方面没有写清楚，因此[也有别人被坑过][3]。（后来发现，通过 `sudo service mongod start` 方式启动会默认使用该配置文件）要想让配置生效，必须手动指定配置文件路径，即：

	mongod --storageEngine wiredTiger --replSet fbt -f /etc/mongod.conf

不过这里不打算用这种方式。既然没有 `/data/db` 这个路径，创建一个不就好了？

	sudo mkdir -p /data/db

继续执行命令1，继续报错  

```
2015-04-23T15:17:49.274+0800 E NETWORK  [initandlisten] Failed to unlink socket file /tmp/mongodb-27017.sock errno:1 Operation not permitted
2015-04-23T15:17:49.274+0800 I -        [initandlisten] Fatal Assertion 28578
2015-04-23T15:17:49.274+0800 I -        [initandlisten]

***aborting after fassert() failure
```
既然它删不掉 `/tmp/mongodb-27017.sock`，那我手动删。删完之后再次执行命令1：

```
2015-04-23T15:19:44.972+0800 I STORAGE  [initandlisten] exception in initAndListen: 98 Unable to create/open lock file: /data/db/mongod.lock errno:13 Permission denied Is a mongod instance already running?, terminating
2015-04-23T15:19:44.972+0800 I CONTROL  [initandlisten] dbexit:  rc: 100
```

这个是可以预期的权限问题，因为我们是拿 `sudo` 创建的 `/data/db` 文件夹，而执行的时候没有用 `sudo`。万能的 [stackoverflow][2] 告诉我们要这样做：

	sudo chown -R `id -u` /data/db

执行之后当前用户就可以在 `/data/db` 创建文件了。再次执行命令1

```
fbt@ebs-39239:~/mongodb$ mongod --storageEngine wiredTiger --replSet fbt
2015-04-23T15:29:38.158+0800 I STORAGE  [initandlisten] wiredtiger_open config: create,cache_size=1G,session_max=20000,eviction=(threads_max=4),statistics=(fast),log=(enabled=true,archive=true,path=journal,compressor=snappy),checkpoint=(wait=60,log_size=2GB),statistics_log=(wait=0),
2015-04-23T15:29:38.264+0800 I CONTROL  [initandlisten] MongoDB starting : pid=30024 port=27017 dbpath=/data/db 64-bit host=ebs-39239
2015-04-23T15:29:38.265+0800 I CONTROL  [initandlisten]
2015-04-23T15:29:38.265+0800 I CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/defrag is 'always'.
2015-04-23T15:29:38.265+0800 I CONTROL  [initandlisten] **        We suggest setting it to 'never'
2015-04-23T15:29:38.265+0800 I CONTROL  [initandlisten]
2015-04-23T15:29:38.265+0800 I CONTROL  [initandlisten] db version v3.0.2
2015-04-23T15:29:38.265+0800 I CONTROL  [initandlisten] git version: 6201872043ecbbc0a4cc169b5482dcf385fc464f
2015-04-23T15:29:38.265+0800 I CONTROL  [initandlisten] OpenSSL version: OpenSSL 1.0.1 14 Mar 2012
2015-04-23T15:29:38.265+0800 I CONTROL  [initandlisten] build info: Linux ip-10-111-179-219 3.2.0-36-virtual #57-Ubuntu SMP Tue Jan 8 22:04:49 UTC 2013 x86_64 BOOST_LIB_VERSION=1_49
2015-04-23T15:29:38.265+0800 I CONTROL  [initandlisten] allocator: tcmalloc
2015-04-23T15:29:38.265+0800 I CONTROL  [initandlisten] options: { replication: { replSet: "fbt" }, storage: { engine: "wiredTiger" } }
2015-04-23T15:29:38.334+0800 I REPL     [initandlisten] Did not find local replica set configuration document at startup;  NoMatchingDocument Did not find replica set configuration document in local.system.replset
2015-04-23T15:29:38.351+0800 I NETWORK  [initandlisten] waiting for connections on port 27017
```

这次终于 OK 了。

如果直接执行，最好是单独开一个 screen。如果不想开 screen，则应当加上 `--fork` 参数把 mongod放到后台去。我采取开 screen 的方式。退出 screen，执行 `ps aux | grep mongo` 观察 mongod 是否成功启动：

```
fbt      29783  0.0  0.0  29476  1676 ?        Ss   14:47   0:00 SCREEN -S mongodb
fbt      30042  3.7  1.7 209004 35496 pts/1    Sl+  15:31   0:00 mongod --storageEngine wiredTiger --replSet fbt
fbt      30059  0.0  0.0  11700   944 pts/0    S+   15:31   0:00 grep --color=auto mongo
```
可以看到，mongod 已经按照之前给的参数在运行了。


[1]: http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/
[2]: http://stackoverflow.com/questions/15229412/unable-to-create-open-lock-file-data-mongod-lock-errno13-permission-denied
[3]: http://stackoverflow.com/questions/12568997/mongodb-not-using-etc-mongodb-conf-after-i-changed-dbpath