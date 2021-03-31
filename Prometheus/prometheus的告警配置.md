新建钉钉群和机器人，在机器人配置里配置ip白名单，三个安全配置里最少要配置一个，钉钉机器人才能发送消息。

下载告警插件
wget https://github.com/prometheus/alertmanager/releases/download/v0.19.0/alertmanager-0.19.0.linux-amd64.tar.gz

下载钉钉插件

wget https://github.com/timonwong/prometheus-webhook-dingtalk/releases/download/v0.3.0/prometheus-webhook-dingtalk-0.3.0.linux-amd64.tar.gz

钉钉机器人的webhook-token

```
https://oapi.dingtalk.com/robot/send?access_token=efbdbc8323e90478c4264042ef2d2ea338dd5b8937a8c69506058d8a3c643c5b

https://oapi.dingtalk.com/robot/send?access_token=8cc160d1dbda303902af77603cae50d74bbfbb474c523eeb82e0ea008c07839e    
```

测试钉钉机器人

```
curl -H "Content-Type: application/json" -d '{"msgtype":"text","text":{"content":"prometheus alert test"}}' https://oapi.dingtalk.com/robot/send?access_token=8cc160d1dbda303902af77603cae50d74bbfbb474c523eeb82e0ea008c07839e

curl -H "Content-Type: application/json" -d '{"msgtype":"text","text":{"content":"prometheus alert test"}}' https://oapi.dingtalk.com/robot/send?access_token=efbdbc8323e90478c4264042ef2d2ea338dd5b8937a8c69506058d8a3c643c5b
```

钉钉群会收到机器人发的消息



**把alertmanager加到服务里。**

```
cat >/lib/systemd/system/alertmanager.service<<\EOF

[Unit]

Description=Prometheus: the alerting system

Documentation=http://prometheus.io/docs/

After=prometheus.service



[Service]

ExecStart=/usr/local/Prometheus/alertmanager/alertmanager --config.file=/usr/local/Prometheus/alertmanager/alertmanager.yml

Restart=always

StartLimitInterval=0

RestartSec=10



[Install]

WantedBy=multi-user.target

EOF
```



**把钉钉加到服务里**

```
cat > /etc/systemd/system/prometheus-webhook-dingtalk.service<<\EOF

[Unit]

Description=prometheus-webhook-dingtalk

After=network-online.target



[Service]

Restart=on-failure

ExecStart=/usr/local/bin/prometheus-webhook-dingtalk \

     --ding.profile=ops_dingding=https://oapi.dingtalk.com/robot/send?access_token=efbdbc8323e90478c4264042ef2d2ea338dd5b8937a8c69506058d8a3c643c5b \

     --ding.profile=info_dingding=https://oapi.dingtalk.com/robot/send?access_token=8cc160d1dbda303902af77603cae50d74bbfbb474c523eeb82e0ea008c07839e     



[Install]

WantedBy=multi-user.target

EOF
```
systemctl daemon-reload
systemctl stop prometheus-webhook-dingtalk
systemctl restart prometheus-webhook-dingtalk
systemctl status prometheus-webhook-dingtalk


**prometheus.yml添加配置**



```abap
alerting:
  alertmanagers:
  - static_configs:
    - targets: ["localhost:9093"]
#      - 10.0.0.91:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "/usr/local/Prometheus/rules/*.yml"
```



**alertmanager.yml配置**

```
global:
  resolve_timeout: 5m # 处理超时时间，默认为5min

# 定义路由树信息
route:
  group_by: [alertname]  # 报警分组依据
  receiver: ops_notify   # 设置默认接收人
  group_wait: 30s        # 最初即第一次等待多久时间发送一组警报的通知
  group_interval: 60s    # 在发送新警报前的等待时间
  repeat_interval: 1h    # 重复发送告警时间。默认1h
  routes:

# 定义基础告警接收者
receivers:
- name: ops_notify
  webhook_configs:
  - url: http://localhost:8060/dingtalk/ops_dingding/send 
    send_resolved: true  # 警报被解决之后是否通知
```



**在rules文件夹下添加配置**

**record-rules.yml**

**注意job!="node-exporter"  不支持匹配正则*  支持 = 或 !=**

```
groups:
  - name: node-exporter-record
    rules:
    - expr: up{job!="node-exporter"}
      record: node_exporter:up
      labels:
        desc: "节点是否在线, 在线1,不在线0"
        unit: " "
        job: "node-exporter"
    - expr: time() - node_boot_time_seconds{}
      record: node_exporter:node_uptime
      labels:
        desc: "节点的运行时间"
        unit: "s"
        job: "node-exporter"
##############################################################################################
#                              cpu                                                           #
    - expr: (1 - avg by (environment,instance) (irate(node_cpu_seconds_total{job!="node-exporter",mode="idle"}[5m])))  * 100
      record: node_exporter:cpu:total:percent
      labels:
        desc: "节点的cpu总消耗百分比"
        unit: "%"
        job: "node-exporter"

    - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job!="node-exporter",mode="idle"}[5m])))  * 100
      record: node_exporter:cpu:idle:percent
      labels:
        desc: "节点的cpu idle百分比"
        unit: "%"
        job: "node-exporter"

    - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job!="node-exporter",mode="iowait"}[5m])))  * 100
      record: node_exporter:cpu:iowait:percent
      labels:
        desc: "节点的cpu iowait百分比"
        unit: "%"
        job: "node-exporter"


    - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job!="node-exporter",mode="system"}[5m])))  * 100
      record: node_exporter:cpu:system:percent
      labels:
        desc: "节点的cpu system百分比"
        unit: "%"
        job: "node-exporter"

    - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job!="node-exporter",mode="user"}[5m])))  * 100
      record: node_exporter:cpu:user:percent
      labels:
        desc: "节点的cpu user百分比"
        unit: "%"
        job: "node-exporter"

    - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job!="node-exporter",mode=~"softirq|nice|irq|steal"}[5m])))  * 100
      record: node_exporter:cpu:other:percent
      labels:
        desc: "节点的cpu 其他的百分比"
        unit: "%"
        job: "node-exporter"
##############################################################################################


##############################################################################################
#                                    memory                                                  #
    - expr: node_memory_MemTotal_bytes{job!="node-exporter"}
      record: node_exporter:memory:total
      labels:
        desc: "节点的内存总量"
        unit: byte
        job: "node-exporter"

    - expr: node_memory_MemFree_bytes{job!="node-exporter"}
      record: node_exporter:memory:free
      labels:
        desc: "节点的剩余内存量"
        unit: byte
        job: "node-exporter"

    - expr: node_memory_MemTotal_bytes{job!="node-exporter"} - node_memory_MemFree_bytes{job!="node-exporter"}
      record: node_exporter:memory:used
      labels:
        desc: "节点的已使用内存量"
        unit: byte
        job: "node-exporter"

    - expr: node_memory_MemTotal_bytes{job!="node-exporter"} - node_memory_MemAvailable_bytes{job!="node-exporter"}
      record: node_exporter:memory:actualused
      labels:
        desc: "节点用户实际使用的内存量"
        unit: byte
        job: "node-exporter"

    - expr: (1-(node_memory_MemAvailable_bytes{job!="node-exporter"} / (node_memory_MemTotal_bytes{job!="node-exporter"})))* 100
      record: node_exporter:memory:used:percent
      labels:
        desc: "节点的内存使用百分比"
        unit: "%"
        job: "node-exporter"

    - expr: ((node_memory_MemAvailable_bytes{job!="node-exporter"} / (node_memory_MemTotal_bytes{job!="node-exporter"})))* 100
      record: node_exporter:memory:free:percent
      labels:
        desc: "节点的内存剩余百分比"
        unit: "%"
        job: "node-exporter"
##############################################################################################
#                                   load                                                     #
    - expr: sum by (instance) (node_load1{job!="node-exporter"})
      record: node_exporter:load:load1
      labels:
        desc: "系统1分钟负载"
        unit: " "
        job: "node-exporter"

    - expr: sum by (instance) (node_load5{job!="node-exporter"})
      record: node_exporter:load:load5
      labels:
        desc: "系统5分钟负载"
        unit: " "
        job: "node-exporter"

    - expr: sum by (instance) (node_load15{job!="node-exporter"})
      record: node_exporter:load:load15
      labels:
        desc: "系统15分钟负载"
        unit: " "
        job: "node-exporter"

##############################################################################################
#                                 disk                                                       #
    - expr: node_filesystem_size_bytes{job!="node-exporter" ,fstype=~"ext4|xfs"}
      record: node_exporter:disk:usage:total
      labels:
        desc: "节点的磁盘总量"
        unit: byte
        job: "node-exporter"

    - expr: node_filesystem_avail_bytes{job!="node-exporter",fstype=~"ext4|xfs"}
      record: node_exporter:disk:usage:free
      labels:
        desc: "节点的磁盘剩余空间"
        unit: byte
        job: "node-exporter"

    - expr: node_filesystem_size_bytes{job!="node-exporter",fstype=~"ext4|xfs"} - node_filesystem_avail_bytes{job!="node-exporter",fstype=~"ext4|xfs"}
      record: node_exporter:disk:usage:used
      labels:
        desc: "节点的磁盘使用的空间"
        unit: byte
        job: "node-exporter"

    - expr:  (1 - node_filesystem_avail_bytes{job!="node-exporter",fstype=~"ext4|xfs"} / node_filesystem_size_bytes{job!="node-exporter",fstype=~"ext4|xfs"}) * 100
      record: node_exporter:disk:used:percent
      labels:
        desc: "节点的磁盘的使用百分比"
        unit: "%"
        job: "node-exporter"

    - expr: irate(node_disk_reads_completed_total{job!="node-exporter"}[1m])
      record: node_exporter:disk:read:count:rate
      labels:
        desc: "节点的磁盘读取速率"
        unit: "次/秒"
        job: "node-exporter"

    - expr: irate(node_disk_writes_completed_total{job!="node-exporter"}[1m])
      record: node_exporter:disk:write:count:rate
      labels:
        desc: "节点的磁盘写入速率"
        unit: "次/秒"
        job: "node-exporter"

    - expr: (irate(node_disk_written_bytes_total{job!="node-exporter"}[1m]))/1024/1024
      record: node_exporter:disk:read:mb:rate
      labels:
        desc: "节点的设备读取MB速率"
        unit: "MB/s"
        job: "node-exporter"

    - expr: (irate(node_disk_read_bytes_total{job!="node-exporter"}[1m]))/1024/1024
      record: node_exporter:disk:write:mb:rate
      labels:
        desc: "节点的设备写入MB速率"
        unit: "MB/s"
        job: "node-exporter"

##############################################################################################
#                                filesystem                                                  #
    - expr:   (1 -node_filesystem_files_free{job!="node-exporter",fstype=~"ext4|xfs"} / node_filesystem_files{job!="node-exporter",fstype=~"ext4|xfs"}) * 100
      record: node_exporter:filesystem:used:percent
      labels:
        desc: "节点的inode的剩余可用的百分比"
        unit: "%"
        job: "node-exporter"
#############################################################################################
#                                filefd                                                     #
    - expr: node_filefd_allocated{job!="node-exporter"}
      record: node_exporter:filefd_allocated:count
      labels:
        desc: "节点的文件描述符打开个数"
        unit: "%"
        job: "node-exporter"

    - expr: node_filefd_allocated{job!="node-exporter"}/node_filefd_maximum{job!="node-exporter"} * 100
      record: node_exporter:filefd_allocated:percent
      labels:
        desc: "节点的文件描述符打开百分比"
        unit: "%"
        job: "node-exporter"

#############################################################################################
#                                network                                                    #
    - expr: avg by (environment,instance,device) (irate(node_network_receive_bytes_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
      record: node_exporter:network:netin:bit:rate
      labels:
        desc: "节点网卡eth0每秒接收的比特数"
        unit: "bit/s"
        job: "node-exporter"

    - expr: avg by (environment,instance,device) (irate(node_network_transmit_bytes_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
      record: node_exporter:network:netout:bit:rate
      labels:
        desc: "节点网卡eth0每秒发送的比特数"
        unit: "bit/s"
        job: "node-exporter"

    - expr: avg by (environment,instance,device) (irate(node_network_receive_packets_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
      record: node_exporter:network:netin:packet:rate
      labels:
        desc: "节点网卡每秒接收的数据包个数"
        unit: "个/秒"
        job: "node-exporter"

    - expr: avg by (environment,instance,device) (irate(node_network_transmit_packets_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
      record: node_exporter:network:netout:packet:rate
      labels:
        desc: "节点网卡发送的数据包个数"
        unit: "个/秒"
        job: "node-exporter"

    - expr: avg by (environment,instance,device) (irate(node_network_receive_errs_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
      record: node_exporter:network:netin:error:rate
      labels:
        desc: "节点设备驱动器检测到的接收错误包的数量"
        unit: "个/秒"
        job: "node-exporter"

    - expr: avg by (environment,instance,device) (irate(node_network_transmit_errs_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
      record: node_exporter:network:netout:error:rate
      labels:
        desc: "节点设备驱动器检测到的发送错误包的数量"
        unit: "个/秒"
        job: "node-exporter"

    - expr: node_tcp_connection_states{job!="node-exporter", state="established"}
      record: node_exporter:network:tcp:established:count
      labels:
        desc: "节点当前established的个数"
        unit: "个"
        job: "node-exporter"

    - expr: node_tcp_connection_states{job!="node-exporter", state="time_wait"}
      record: node_exporter:network:tcp:timewait:count
      labels:
        desc: "节点timewait的连接数"
        unit: "个"
        job: "node-exporter"

    - expr: sum by (environment,instance) (node_tcp_connection_states{job!="node-exporter"})
      record: node_exporter:network:tcp:total:count
      labels:
        desc: "节点tcp连接总数"
        unit: "个"
        job: "node-exporter"

#############################################################################################
#                                process                                                    #
    - expr: node_processes_state{state="Z"}
      record: node_exporter:process:zoom:total:count
      labels:
        desc: "节点当前状态为zoom的个数"
        unit: "个"
        job: "node-exporter"
#############################################################################################
#                                other                                                    #
    - expr: abs(node_timex_offset_seconds{job!="node-exporter"})
      record: node_exporter:time:offset
      labels:
        desc: "节点的时间偏差"
        unit: "s"
        job: "node-exporter"

#############################################################################################

    - expr: count by (instance) ( count by (instance,cpu) (node_cpu_seconds_total{ mode='system'}) )
      record: node_exporter:cpu:count
```



**添加对应的alert-rules.yml**

```
groups:
  - name: node-exporter-alert
    rules:
    - alert: node-exporter-down
      expr: node_exporter:up == 0
      for: 1m
      labels:
        severity: 'critical'
      annotations:
        summary: "instance: {{ $labels.instance }} 宕机了"
        description: "instance: {{ $labels.instance }} \n- job: {{ $labels.job }} 关机了， 时间已经1分钟了。"
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"



    - alert: node-exporter-cpu-high
      expr:  node_exporter:cpu:total:percent > 80
      for: 3m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} cpu 使用率高于 {{ $value }}"
        description: "instance: {{ $labels.instance }} \n- job: {{ $labels.job }} CPU使用率已经持续三分钟高过80% 。"
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"

    - alert: node-exporter-cpu-iowait-high
      expr:  node_exporter:cpu:iowait:percent >= 12
      for: 3m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} cpu iowait 使用率高于 {{ $value }}"
        description: "instance: {{ $labels.instance }} \n- job: {{ $labels.job }} cpu iowait使用率已经持续三分钟高过12%"
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"


    - alert: node-exporter-load-load1-high
      expr:  (node_exporter:load:load1) > (node_exporter:cpu:count) * 1.2
      for: 3m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} load1 使用率高于 {{ $value }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"


    - alert: node-exporter-memory-high
      expr:  node_exporter:memory:used:percent > 85
      for: 3m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} memory 使用率高于 {{ $value }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"


    - alert: node-exporter-disk-high
      expr:  node_exporter:disk:used:percent > 70
      for: 10m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} disk 使用率高于 {{ $value }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"


    - alert: node-exporter-disk-read:count-high
      expr:  node_exporter:disk:read:count:rate > 2000
      for: 2m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} iops read 使用率高于 {{ $value }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"


    - alert: node-exporter-disk-write-count-high
      expr:  node_exporter:disk:write:count:rate > 2000
      for: 2m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} iops write 使用率高于 {{ $value }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"





    - alert: node-exporter-disk-read-mb-high
      expr:  node_exporter:disk:read:mb:rate > 60
      for: 2m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 读取字节数 高于 {{ $value }}"
        description: ""
        instance: "{{ $labels.instance }}"
        value: "{{ $value }}"


    - alert: node-exporter-disk-write-mb-high
      expr:  node_exporter:disk:write:mb:rate > 60
      for: 2m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 写入字节数 高于 {{ $value }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"


    - alert: node-exporter-filefd-allocated-percent-high
      expr:  node_exporter:filefd_allocated:percent > 80
      for: 10m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 打开文件描述符 高于 {{ $value }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"


    - alert: node-exporter-network-netin-error-rate-high
      expr:  node_exporter:network:netin:error:rate > 4
      for: 1m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 包进入的错误速率 高于 {{ $value }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"

    - alert: node-exporter-network-netin-packet-rate-high
      expr:  node_exporter:network:netin:packet:rate > 35000
      for: 1m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 包进入速率 高于 {{ $value }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"


    - alert: node-exporter-network-netout-packet-rate-high
      expr:  node_exporter:network:netout:packet:rate > 35000
      for: 1m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 包流出速率 高于 {{ $value }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"


    - alert: node-exporter-network-tcp-total-count-high
      expr:  node_exporter:network:tcp:total:count > 40000
      for: 1m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} tcp连接数量 高于 {{ $value }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"


    - alert: node-exporter-process-zoom-total-count-high
      expr:  node_exporter:process:zoom:total:count > 10
      for: 10m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 僵死进程数量 高于 {{ $value }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"


    - alert: node-exporter-time-offset-high
      expr:  node_exporter:time:offset > 0.03
      for: 2m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} {{ $labels.desc }}  {{ $value }} {{ $labels.unit }}"
        description: ""
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
```



重启prometheus服务

现在已经测试过内存和cpu的报警，正常。



**启动服务加上参数开启热加载**

nohup /usr/local/Prometheus/prometheus --config.file=/usr/local/Prometheus/prometheus.yml --web.enable-lifecycle &



热加载命令

curl -X POST http://127.0.0.1:9090/-/reload


