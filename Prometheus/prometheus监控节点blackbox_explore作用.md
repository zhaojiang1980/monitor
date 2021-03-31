prometheus监控节点blackbox_explore作用



```
提供 http、dns、tcp、icmp 的监控数据采集
Blackbox_exporter 应用场景
HTTP 测试
定义 Request Header 信息
判断 Http status / Http Respones Header / Http Body 内容TCP 测试
业务组件端口状态监听
应用层协议定义与监听ICMP 测试
主机探活机制POST 测试
接口联通性SSL 证书过期时间
https://github.com/prometheus/blackbox_exporter
https://cloud.tencent.com/developer/news/620963
https://lyz-code.github.io/blue-book/devops/prometheus/blackbox_exporter/
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.18.0/blackbox_exporter-0.18.0.linux-amd64.tar.gz
tar -zxvf blackbox_exporter-0.18.0.linux-amd64.tar.gz -C /usr/local/

#启动blackbox
nohup ./blackbox_explore --config.file=./blackbox.yml &

#新增以下配置
######2020 12 28 add  web monitor
  - job_name: 'http_status'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets: ['https://www.pre.supers.io', 'https://m.pre.supers.io']
        labels:
          instance: http_status
          group: web
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 10.0.0.177:9115

#ping check
  - job_name: 'ping_status'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets: ['10.0.0.41','10.0.0.177']
        labels:
          instance: 'ping_status'
          group: 'icmp'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 10.0.0.177:9115

#port check
  - job_name: 'port_status'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets: ['10.0.0.41:8000', '10.0.0.41:8020', '10.0.0.41:8040']
        labels:
          instance: 'port_status'
          group: 'port'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 10.0.0.177:9115
curl prometheus.supers.io/-/reload
1. 登陆Prometheus 模板管理
2. 搜索 9965 import导入即可。
```