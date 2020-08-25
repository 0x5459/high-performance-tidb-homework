# 课程02 作业

## 环境

|IP|CPU|内存|硬盘|OS|备注|
|-|-|-|-|-|-|
|172.19.215.74|8核|16G|SSD180G|Ubuntu 20.04|TIKV&PD|
|172.19.215.75|8核|16G|SSD180G|Ubuntu 20.04|TIKV&PD|
|172.19.215.78|8核|16G|SSD180G|Ubuntu 20.04|TIKV&PD|
|172.19.215.80|8核|16G|SSD40G|Ubuntu 20.04|TIDB|
|172.19.215.81|8核|16G|SSD40G|Ubuntu 20.04|TIDB|
|172.19.215.84|4核|4G|SSD40G|Ubuntu 20.04|监控&施压机|


## 部署

使用 TIUP 部署 ([TIUP 安装文档](https://docs.pingcap.com/zh/tidb/stable/production-deployment-using-tiup))

TIUP 配置:
```ymal
global:
  user: "tidb"
  ssh_port: 22
  deploy_dir: "/tidb-deploy"
  data_dir: "/tidb-data"

pd_servers:
  - host: 172.19.215.78
  - host: 172.19.215.74
  - host: 172.19.215.75

tidb_servers:
  - host: 172.19.215.81
  - host: 172.19.215.80

tikv_servers:
  - host: 172.19.215.78
  - host: 172.19.215.74
  - host: 172.19.215.75

monitoring_servers:
  - host: 172.19.215.84

grafana_servers:
  - host: 172.19.215.84

alertmanager_servers:
  - host: 172.19.215.84
```

部署集群

`tiup cluster deploy tidb-test v4.0.0 ./topology.yaml --user root -p`

启动集群

`tiup cluster start tidb-test`

检查集群状态

`tiup cluster display tidb-test`

```shell
Starting component `cluster`:  display tidb-test
tidb Cluster: tidb-test
tidb Version: v4.0.0
ID                   Role          Host           Ports        OS/Arch       Status  Data Dir                      Deploy Dir
--                   ----          ----           -----        -------       ------  --------                      ----------
172.19.215.84:9093   alertmanager  172.19.215.84  9093/9094    linux/x86_64  Up      /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
172.19.215.84:3000   grafana       172.19.215.84  3000         linux/x86_64  Up      -                             /tidb-deploy/grafana-3000
172.19.215.74:2379   pd            172.19.215.74  2379/2380    linux/x86_64  Up|UI   /tidb-data/pd-2379            /tidb-deploy/pd-2379
172.19.215.75:2379   pd            172.19.215.75  2379/2380    linux/x86_64  Up      /tidb-data/pd-2379            /tidb-deploy/pd-2379
172.19.215.78:2379   pd            172.19.215.78  2379/2380    linux/x86_64  Up|L    /tidb-data/pd-2379            /tidb-deploy/pd-2379
172.19.215.84:9090   prometheus    172.19.215.84  9090         linux/x86_64  Up      /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
172.19.215.80:4000   tidb          172.19.215.80  4000/10080   linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
172.19.215.81:4000   tidb          172.19.215.81  4000/10080   linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
172.19.215.74:20160  tikv          172.19.215.74  20160/20180  linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
172.19.215.75:20160  tikv          172.19.215.75  20160/20180  linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
172.19.215.78:20160  tikv          172.19.215.78  20160/20180  linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160

```


TIDB Dashboard

![TIDB Dashboard](media/lesson02/dashboard-1.png)

Grafana 监控

![Grafana](media/lesson02/grafana-1.png)


使用 HAProxy 代理 TIDB

在 172.19.215.80 中安装 HAPROXY [安装文档](https://docs.pingcap.com/zh/tidb/stable/haproxy-best-practices#%E5%90%AF%E5%8A%A8-haproxy)。

HAProxy 配置
```shell
# /etc/haproxy/haproxy.cfg

global                                     # 全局配置。
   log         127.0.0.1 local2            # 定义全局的 syslog 服务器，最多可以定义两个。
   chroot      /var/lib/haproxy            # 更改当前目录并为启动进程设置超级用户权限，从而提高安全性。
   pidfile     /var/run/haproxy.pid        # 将 HAProxy 进程的 PID 写入 pidfile。
   maxconn     4000                        # 每个 HAProxy 进程所接受的最大并发连接数。
   user        haproxy                     # 同 UID 参数。
   group       haproxy                     # 同 GID 参数，建议使用专用用户组。
   nbproc      40                          # 在后台运行时创建的进程数。在启动多个进程转发请求时，确保该值足够大，保证 HAProxy 不会成为瓶颈。
   daemon                                  # 让 HAProxy 以守护进程的方式工作于后台，等同于命令行参数“-D”的功能。当然，也可以在命令行中用“-db”参数将其禁用。
   stats socket /var/lib/haproxy/stats     # 统计信息保存位置。

defaults                                   # 默认配置。
   log global                              # 日志继承全局配置段的设置。
   retries 2                               # 向上游服务器尝试连接的最大次数，超过此值便认为后端服务器不可用。
   timeout connect  2s                     # HAProxy 与后端服务器连接超时时间。如果在同一个局域网内，可设置成较短的时间。
   timeout client 30000s                   # 客户端与 HAProxy 连接后，数据传输完毕，即非活动连接的超时时间。
   timeout server 30000s                   # 服务器端非活动连接的超时时间。
listen admin_stats                         # frontend 和 backend 的组合体，此监控组的名称可按需进行自定义。
   bind 0.0.0.0:8080                       # 监听端口。
   mode http                               # 监控运行的模式，此处为 `http` 模式。
   option httplog                          # 开始启用记录 HTTP 请求的日志功能。
   maxconn 10                              # 最大并发连接数。
   stats refresh 30s       
                   # 每隔 30 秒自动刷新监控页面。
   stats uri /haproxy                      # 监控页面的 URL。
   stats realm HAProxy                     # 监控页面的提示信息。
   stats auth admin:tidb123abc             # 监控页面的用户和密码，可设置多个用户名。
   stats hide-version                      # 隐藏监控页面上的 HAProxy 版本信息。
   stats  admin if TRUE                    # 手工启用或禁用后端服务器（HAProxy 1.4.9 及之后版本开始支持）。

listen tidb-cluster                        # 配置 database 负载均衡。
   bind 0.0.0.0:3390                       # 浮动 IP 和 监听端口。
   mode tcp                                # HAProxy 要使用第 4 层的传输层。
   balance leastconn                       # 连接数最少的服务器优先接收连接。`leastconn` 建议用于长会话服务，例如 LDAP、SQL、TSE 等，而不是短会话协议，如 HTTP。该算法是动态的，对于启动慢的服务器，服务器权重会在运行中作调整。
   server tidb-1 172.19.215.81:4000 check inter 2000 rise 2 fall 3       # 检测 4000 端口，检测频率为每 2000 毫秒一次。如果 2 次检测为成功，则认为服务器可用；如果 3 次检测为失败，则认为服务器不可用。
   server tidb-2 172.19.215.80:4000 check inter 2000 rise 2 fall 3
```
启动 HAProxy

`haproxy -f /etc/haproxy/haproxy.cfg`

访问 `http://ip:8080/haproxy` 可以查看代理状态
![HAProxy](media/lesson02/haproxy-1.jpg)

至此 TIDB 集群部署完毕。


## 测试

### 1. sysbench

安装 sysbench 
```shell
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash

sudo apt -y install sysbench
```

sysbench 配置文件
```
mysql-host=172.19.215.80
mysql-port=3390
mysql-user=root
mysql-password=
mysql-db=sbtest
time=600
threads=16
report-interval=10
db-driver=mysql
```

设置乐观事务模式

`mysql -u root -h 172.19.215.80 -P 3390`
```
mysql> set global tidb_disable_txn_auto_retry = off;
mysql> set global tidb_txn_mode="optimistic";
```

导入数据

`nohup sysbench --config-file=config oltp_point_select --threads=128 --tables=32 --table-size=500000 prepare 2>&1 &`

Point select 测试

`sysbench --config-file=config oltp_point_select --threads=256 --tables=32 --table-size=500000 run`

```
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 256
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...


Threads started!


[ 10s ] thds: 256 tps: 91863.72 qps: 91863.72 (r/w/o: 91863.72/0.00/0.00) lat (ms,95%): 8.13 err/s: 0.00 reconn/s: 0.00
...
...
...
[ 600s ] thds: 256 tps: 95278.57 qps: 95278.57 (r/w/o: 95278.57/0.00/0.00) lat (ms,95%): 8.43 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            56378716
        write:                           0
        other:                           0
        total:                           56378716
    transactions:                        56378716 (93954.66 per sec.)
    queries:                             56378716 (93954.66 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.0614s
    total number of events:              56378716

Latency (ms):
         min:                                    0.40
         avg:                                    2.72
         max:                                   59.86
         95th percentile:                        8.43
         sum:                            153564385.49

Threads fairness:
    events (avg/stddev):           220229.3594/109371.77
    execution time (avg/stddev):   599.8609/0.10
```


Update index 测试
`sysbench --config-file=config oltp_update_index --threads=256 --tables=32 --table-size=500000 run`

```
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 256
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 256 tps: 14391.32 qps: 14391.32 (r/w/o: 0.00/14080.48/310.84) lat (ms,95%): 22.28 err/s: 0.00 reconn/s: 0.00
...
...
...
[ 600s ] thds: 256 tps: 4593.93 qps: 4593.93 (r/w/o: 0.00/4494.93/99.00) lat (ms,95%): 308.84 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            0
        write:                           6723627
        other:                           148656
        total:                           6872283
    transactions:                        6872283 (11450.85 per sec.)
    queries:                             6872283 (11450.85 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.1533s
    total number of events:              6872283

Latency (ms):
         min:                                    0.56
         avg:                                   22.35
         max:                                 3584.85
         95th percentile:                       30.26
         sum:                            153615313.12

Threads fairness:
    events (avg/stddev):           26844.8555/381.79
    execution time (avg/stddev):   600.0598/0.05
```

Read-only 测试

`sysbench --config-file=config oltp_read_only --threads=256 --tables=32 --table-size=500000 run`

```
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 256
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 256 tps: 2415.04 qps: 38827.36 (r/w/o: 33974.28/0.00/4853.08) lat (ms,95%): 150.29 err/s: 0.00 reconn/s: 0.00
...
...
...
[ 600s ] thds: 256 tps: 2420.75 qps: 38750.34 (r/w/o: 33909.25/0.00/4841.09) lat (ms,95%): 153.02 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            20253366
        write:                           0
        other:                           2893338
        total:                           23146704
    transactions:                        1446669 (2410.36 per sec.)
    queries:                             23146704 (38565.79 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.1858s
    total number of events:              1446669

Latency (ms):
         min:                                   12.81
         avg:                                  106.18
         max:                                  471.25
         95th percentile:                      153.02
         sum:                            153611083.10

Threads fairness:
    events (avg/stddev):           5651.0508/875.76
    execution time (avg/stddev):   600.0433/0.04
```


