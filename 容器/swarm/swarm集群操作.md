docker swarm集群

1、docker安装

阿里云安装
[root@locahost ~]# wget -P /etc/yum.repos.d/  https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
[root@locahost ~]# yum -y install docker-ce
官方安装
[root@locahost ~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
[root@locahost ~]# yum -y install docker-ce

Docker启动
systemctl start docker.service
systemctl enable docker

Docker检查
安装完成之后，执行下面命令检查一下安装是否正常
docker version
docker info

2、compose安装

一键编排工具（docker-compose）
1)
到地址： https://github.com/docker/compose/releases  查看最新版本安装

curl -L https://github.com/docker/compose/releases/download/1.14.0/docker-compose-`uname -s`-`uname -m` > /usr/bin/docker-compose
chmod +x /usr/bin/docker-compose

2)Install using pip

Compose can be installed from pypi using pip. If you install using pip, we recommend that you use a virtualenv because many operating systems have python system packages that conflict with docker-compose dependencies. See the virtualenv tutorial to get started.

pip install docker-compose
if you are not using virtualenv,

sudo pip install docker-compose
pip version 6.0 or greater is required.

3、修改默认存储路径
Q： /var/lib/docker 使用 / 根分区， 空间不够
A： 修改默认存储路径，
          1） 修改文件 /lib/systemd/system/docker.service 中  
                            ExecStart=/usr/bin/dockerd
                改为
                            ExecStart=/usr/bin/dockerd --graph /data/var/lib/docker
          2） 重启docker服务

或者修改/etc/docker/daemon.json文件
# cat /etc/docker/daemon.json
{
  "graph": "/data/docker",
  "registry-mirrors": ["https://zvz7k9mh.mirror.aliyuncs.com"]
}


2、swarm安装
192.168.203.230  master
192.168.201.37   worker
192.168.201.125  worker
1)master上面init
[root@LOCAL-203-230 ~]# docker swarm init --advertise-addr 192.168.203.230
Swarm initialized: current node (unf8fy65j8lho6qc8lpevwwcu) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-1kdygagmiiky90yjay810l0b2ngy673bwtokg17uixfvwjq7hj-9md0lm3jxyq6y3l08k9fbk4my 192.168.203.230:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
2)worker加入集群
通过上面的提示加入

[root@VM-201-125 ~]# docker swarm join --token SWMTKN-1-1kdygagmiiky90yjay810l0b2ngy673bwtokg17uixfvwjq7hj-9md0lm3jxyq6y3l08k9fbk4my 192.168.203.230:2377
This node joined a swarm as a worker.
[root@vm_201 ~]# docker swarm join --token SWMTKN-1-1kdygagmiiky90yjay810l0b2ngy673bwtokg17uixfvwjq7hj-9md0lm3jxyq6y3l08k9fbk4my 192.168.203.230:2377
This node joined a swarm as a worker.

这样我们就得到了上3个管理节点组成的集群，这样的集群任意宕机1台都不影响业务正常运行。

3)状态查看
docker info查看状态
[root@vm_201 ~]# docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 17.06.0-ce
Swarm: active
 NodeID: wnula7v11l3uhaqh78cqqz46t
 Is Manager: false
 Node Address: 192.168.201.37
 Manager Addresses:
  192.168.203.230:2377
Docker Root Dir: /data/docker
Debug Mode (client): false
Debug Mode (server): false
Registry: https://index.docker.io/v1/
Experimental: false
Insecure Registries:
 127.0.0.0/8
Registry Mirrors:
 https://zvz7k9mh.mirror.aliyuncs.com/
Live Restore Enabled: false


[root@LOCAL-203-230 ~]# docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 17.06.0-ce
Swarm: active
 NodeID: unf8fy65j8lho6qc8lpevwwcu
 Is Manager: true
 ClusterID: 14jmu6yqx1rompjxkowxgwu7m
 Managers: 1
 Nodes: 3
 Orchestration:
  Task History Retention Limit: 5
 Raft:
  Snapshot Interval: 10000
  Number of Old Snapshots to Retain: 0
  Heartbeat Tick: 1
  Election Tick: 3
 Dispatcher:
  Heartbeat Period: 5 seconds
 CA Configuration:
  Expiry Duration: 3 months
  Force Rotate: 0
 Root Rotation In Progress: false
 Node Address: 192.168.203.230
 Manager Addresses:
  192.168.203.230:2377

[root@VM-201-125 ~]# docker info
Containers: 22
 Running: 16
 Paused: 0
 Stopped: 6
Images: 229
Server Version: 17.06.0-ce
Swarm: active
 NodeID: oxnf49auk27urz5k3ir2zdp9q
 Is Manager: false
 Node Address: 192.168.201.125
 Manager Addresses:
  192.168.203.230:2377

master上面运行：
[root@LOCAL-203-230 ~]# docker node ls
ID                            HOSTNAME                  STATUS              AVAILABILITY        MANAGER STATUS
oxnf49auk27urz5k3ir2zdp9q     VM-201-125                Ready               Active              
unf8fy65j8lho6qc8lpevwwcu *   LOCAL-203-230.boyaa.com   Ready               Active              Leader
wnula7v11l3uhaqh78cqqz46t     vm_201.37.boyaa.com       Ready               Active

4)部署一个service
master节点上面运行
[root@LOCAL-203-230 ~]# docker service create --replicas 1 --name helloworld alpine ping docker.com
4mxwhlgjai9piu2e4tek0hhdm
Since --detach=false was not specified, tasks will be created in the background.
In a future release, --detach=false will become the default.
查看
[root@LOCAL-203-230 ~]# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
4mxwhlgjai9p        helloworld          replicated          1/1                 alpine:latest

[root@LOCAL-203-230 ~]# docker service  ps  4mxwhlgjai9p
ID                  NAME                IMAGE               NODE                      DESIRED STATE       CURRENT STATE                ERROR               PORTS
48ol7m380awx        helloworld.1        alpine:latest       LOCAL-203-230.boyaa.com   Running             Running about a minute ago

删除一个服务
[root@LOCAL-203-230 ~]# docker service rm 4mxwhlgjai9p
4mxwhlgjai9p
[root@LOCAL-203-230 ~]# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
[root@LOCAL-203-230 ~]#

运行一个replicas为3的服务
[root@LOCAL-203-230 ~]# docker service create --replicas 3 --name helloworld alpine ping docker.com
f8xmjx5gb1s2lgcyiqrwk46d2
Since --detach=false was not specified, tasks will be created in the background.
In a future release, --detach=false will become the default.
[root@LOCAL-203-230 ~]# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
f8xmjx5gb1s2        helloworld          replicated          3/3                 alpine:latest       
[root@LOCAL-203-230 ~]# docker service ps f8xmjx5gb1s2
ID                  NAME                IMAGE               NODE                      DESIRED STATE       CURRENT STATE            ERROR               PORTS
j2ynnz93dt34        helloworld.1        alpine:latest       vm_201.37.boyaa.com       Running             Running 17 seconds ago                       
yg48df58kpiw        helloworld.2        alpine:latest       VM-201-125                Running             Running 11 seconds ago                       
dv3a9esgknoz        helloworld.3        alpine:latest       LOCAL-203-230.boyaa.com   Running             Running 19 seconds ago
查看日志
[root@LOCAL-203-230 ~]# docker service logs f8xmjx5gb1s2lgcyiqrwk46d2
helloworld.3.dv3a9esgknoz@LOCAL-203-230.boyaa.com    | PING docker.com (52.2.237.188): 56 data bytes
helloworld.1.j2ynnz93dt34@vm_201.37.boyaa.com    | PING docker.com (34.196.237.26): 56 data bytes
helloworld.2.yg48df58kpiw@VM-201-125    | PING docker.com (52.2.237.188): 56 data bytes


查看服务的详细信息
[root@LOCAL-203-230 ~]# docker service inspect --pretty helloworld

ID:		f8xmjx5gb1s2lgcyiqrwk46d2
Name:		helloworld
Service Mode:	Replicated
 Replicas:	3
Placement:
UpdateConfig:
 Parallelism:	1
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:	1
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:		alpine:latest@sha256:1072e499f3f655a032e88542330cf75b02e7bdf673278f701d7ba61629ee3ebe
 Args:		ping docker.com
Resources:
Endpoint Mode:	vip

不加--pretty返回json格式


调整swarm里面的服务(Scale the service in the swarm)

[root@LOCAL-203-230 ~]# docker service scale f8xmjx5gb1s2lgcyiqrwk46d2=2
f8xmjx5gb1s2lgcyiqrwk46d2 scaled to 2
[root@LOCAL-203-230 ~]# docker service ps f8xmjx5gb1s2lgcyiqrwk46d2
ID                  NAME                IMAGE               NODE                  DESIRED STATE       CURRENT STATE           ERROR               PORTS
j2ynnz93dt34        helloworld.1        alpine:latest       vm_201.37.boyaa.com   Running             Running 5 minutes ago                       
yg48df58kpiw        helloworld.2        alpine:latest       VM-201-125            Running             Running 5 minutes ago

[root@LOCAL-203-230 ~]# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
f8xmjx5gb1s2        helloworld          replicated          2/2                 alpine:latest       
[root@LOCAL-203-230 ~]# docker service inspect --pretty helloworld

ID:		f8xmjx5gb1s2lgcyiqrwk46d2
Name:		helloworld
Service Mode:	Replicated
 Replicas:	2
Placement:
UpdateConfig:
 Parallelism:	1
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:	1
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:		alpine:latest@sha256:1072e499f3f655a032e88542330cf75b02e7bdf673278f701d7ba61629ee3ebe
 Args:		ping docker.com
Resources:
Endpoint Mode:	vip


滚动更新应用(Apply rolling updates to a service)


[root@LOCAL-203-230 ~]#  docker service create \
>   --replicas 3 \
>   --name redis \
>   --update-delay 10s \
>   redis:3.0.6
xjqsl9w96wdidshc9oy5q2at1
Since --detach=false was not specified, tasks will be created in the background.
In a future release, --detach=false will become the default.

[root@LOCAL-203-230 ~]# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
xjqsl9w96wdi        redis               replicated          0/3                 redis:3.0.6         
[root@LOCAL-203-230 ~]# docker service ps xjqsl9w96wdi
ID                  NAME                IMAGE               NODE                      DESIRED STATE       CURRENT STATE              ERROR               PORTS
gd2aw35r91ki        redis.1             redis:3.0.6         VM-201-125                Running             Preparing 14 seconds ago                       
yn25n8dhznkz        redis.2             redis:3.0.6         LOCAL-203-230.boyaa.com   Running             Preparing 14 seconds ago                       
iib3q1zygut2        redis.3             redis:3.0.6         vm_201.37.boyaa.com       Running             Preparing 14 seconds ago


[root@LOCAL-203-230 ~]# docker service inspect --pretty redis

ID:		xjqsl9w96wdidshc9oy5q2at1
Name:		redis
Service Mode:	Replicated
 Replicas:	3
Placement:
UpdateConfig:
 Parallelism:	1
 Delay:		10s
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:	1
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:		redis:3.0.6@sha256:6a692a76c2081888b589e26e6ec835743119fe453d67ecf03df7de5b73d69842
Resources:
Endpoint Mode:	vip


[root@LOCAL-203-230 ~]# docker service update --image redis:3.0.7 redis
redis
Since --detach=false was not specified, tasks will be updated in the background.
In a future release, --detach=false will become the default.
[root@LOCAL-203-230 ~]# docker service inspect --pretty redis

ID:		xjqsl9w96wdidshc9oy5q2at1
Name:		redis
Service Mode:	Replicated
 Replicas:	3
UpdateStatus:
 State:		updating
 Started:	3 seconds ago
 Message:	update in progress
Placement:
UpdateConfig:
 Parallelism:	1
 Delay:		10s
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:	1
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:		redis:3.0.7@sha256:730b765df9fe96af414da64a2b67f3a5f70b8fd13a31e5096fee4807ed802e20
Resources:
Endpoint Mode:	vip


[root@LOCAL-203-230 ~]# docker service ps redis
ID                  NAME                IMAGE               NODE                      DESIRED STATE       CURRENT STATE                     ERROR               PORTS
vb012yroapm9        redis.1             redis:3.0.7         VM-201-125                Running             Running 34 seconds ago                                
gd2aw35r91ki         \_ redis.1         redis:3.0.6         VM-201-125                Shutdown            Shutdown 45 seconds ago                               
yn25n8dhznkz        redis.2             redis:3.0.6         LOCAL-203-230.boyaa.com   Running             Running 16 minutes ago                                
39if3zdikml0        redis.3             redis:3.0.7         vm_201.37.boyaa.com       Running             Starting less than a second ago                       
iib3q1zygut2         \_ redis.3         redis:3.0.6         vm_201.37.boyaa.com       Shutdown            Shutdown 22 seconds ago

[root@LOCAL-203-230 ~]# docker service ps redis
ID                  NAME                IMAGE               NODE                      DESIRED STATE       CURRENT STATE                 ERROR               PORTS
vb012yroapm9        redis.1             redis:3.0.7         VM-201-125                Running             Running about a minute ago                        
gd2aw35r91ki         \_ redis.1         redis:3.0.6         VM-201-125                Shutdown            Shutdown about a minute ago                       
upb3k0lmdjbc        redis.2             redis:3.0.7         LOCAL-203-230.boyaa.com   Running             Preparing 38 seconds ago                          
yn25n8dhznkz         \_ redis.2         redis:3.0.6         LOCAL-203-230.boyaa.com   Shutdown            Shutdown 37 seconds ago                           
39if3zdikml0        redis.3             redis:3.0.7         vm_201.37.boyaa.com       Running             Running 48 seconds ago                            
iib3q1zygut2         \_ redis.3         redis:3.0.6         vm_201.37.boyaa.com       Shutdown            Shutdown about a minute ago


退除一个节点(Drain a node on the swarm)

$ docker node ls

ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
1bcef6utixb0l0ca7gxuivsj0    worker2   Ready   Active
38ciaotwjuritcdtn9npbnkuz    worker1   Ready   Active
e216jshn25ckzbvmwlnh5jr3g *  manager1  Ready   Active        Leader


$ docker service create --replicas 3 --name redis --update-delay 10s redis:3.0.6

c5uo6kdmzpon37mgj9mwglcfw

$ docker service ps redis

NAME                               IMAGE        NODE     DESIRED STATE  CURRENT STATE
redis.1.7q92v0nr1hcgts2amcjyqg3pq  redis:3.0.6  manager1 Running        Running 26 seconds
redis.2.7h2l8h3q3wqy5f66hlv9ddmi6  redis:3.0.6  worker1  Running        Running 26 seconds
redis.3.9bg7cezvedmkgg6c8yzvbhwsd  redis:3.0.6  worker2  Running        Running 26 seconds

In this case the swarm manager distributed one task to each node. You may see the tasks distributed differently among the nodes in your environment.


Run docker node update --availability drain <NODE-ID> to drain a node that had a task assigned to it:

docker node update --availability drain worker1

worker1


Inspect the node to check its availability:

$ docker node inspect --pretty worker1

ID:			38ciaotwjuritcdtn9npbnkuz
Hostname:		worker1
Status:
 State:			Ready
 Availability:		Drain
...snip...

The drained node shows Drain for AVAILABILITY.

$ docker service ps redis

NAME                                    IMAGE        NODE      DESIRED STATE  CURRENT STATE           ERROR
redis.1.7q92v0nr1hcgts2amcjyqg3pq       redis:3.0.6  manager1  Running        Running 4 minutes
redis.2.b4hovzed7id8irg1to42egue8       redis:3.0.6  worker2   Running        Running About a minute
 \_ redis.2.7h2l8h3q3wqy5f66hlv9ddmi6   redis:3.0.6  worker1   Shutdown       Shutdown 2 minutes ago
redis.3.9bg7cezvedmkgg6c8yzvbhwsd       redis:3.0.6  worker2   Running        Running 4 minutes

$ docker node update --availability active worker1


Inspect the node to see the updated state:

$ docker node inspect --pretty worker1
worker1
ID:			38ciaotwjuritcdtn9npbnkuz
Hostname:		worker1
Status:
 State:			Ready
 Availability:		Active
...snip...
worker1
