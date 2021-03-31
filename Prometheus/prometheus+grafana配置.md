prometheus+grafana配置


参考	https://blog.rj-bai.com/post/158.html


二、安装go
1、解压安装
tar -C /usr/local/ -xvf go1.11.4.linux-amd64.tar.gz
1
2、配置环境变量
vim /etc/profile

export PATH=$PATH:/usr/local/go/bin

source /etc/profile

3、验证
go version

安装Prometheus

tar -C /usr/local/ -xvf prometheus-2.6.0.linux-amd64.tar.gz
ln -sv /usr/local/prometheus-2.6.0.linux-amd64/ /usr/local/Prometheus

启动
普罗米修斯默认配置文件 vim /usr/local/Prometheus/prometheus.yml

/usr/local/Prometheus/prometheus --config.file=/usr/local/Prometheus/prometheus.yml &

3、验证
浏览器打开IP:9090端口即可打开普罗米修斯自带的监控页面


安装Grafana
普罗米修斯默认的页面可能没有那么直观，我们可以安装grafana使监控看起来更直观

1、安装
wget https://dl.grafana.com/oss/release/grafana-7.0.0-1.x86_64.rpm
sudo yum install grafana-7.0.0-1.x86_64.rpm

2、启动
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable grafana-server.service
sudo /bin/systemctl start grafana-server.service

安装一些插件

/usr/sbin/grafana-cli  plugins install grafana-piechart-panel
/usr/sbin/grafana-cli  plugins install michaeldmoore-annunciator-panel
/usr/sbin/grafana-cli  plugins install michaeldmoore-multistat-panel  
/usr/sbin/grafana-cli  plugins install grafana-clock-panel

浏览器访问IP:3000端口，即可打开grafana页面，默认用户名密码都是admin，初次登录会要求修改默认的登录密码

4、添加prometheus数据源
（1）点击主界面的“Add data source”

（2）选择Prometheus

（3）Dashboards页面选择“Prometheus 2.0 Stats”

（4）Settings页面填写普罗米修斯地址并保存

1、监控linux机器（node-exporter）
https://github.com/prometheus/node_exporter/releases/download/v0.17.0/node_exporter-0.17.0.linux-amd64.tar.gz
（1）被监控的机器安装node-exporter

tar -xvf node_exporter-0.17.0.linux-amd64.tar.gz -C /usr/local/
1
（2）启动node-exporter

/usr/local/node_exporter-0.17.0.linux-amd64/node_exporter &
1
（3）普罗米修斯配置文件添加监控项

vim /usr/local/Prometheus/prometheus.yml
1
默认node-exporter端口为9100

  - job_name: 'dev-211.182'
    static_configs:
    - targets: ['172.31.114.6:9100']
      labels:
        instance: dev-211.182
注意配置文件的json格式的，注意空格，文件对齐。

安装 cadvisor 监控docker 并配置网络

docker run --volume=/:/rootfs:ro -v /etc/localtime:/etc/localtime:ro --volume=/var/run:/var/run:rw   --volume=/sys:/sys:ro   --volume=/var/lib/docker/:/var/lib/docker:ro   --volume=/dev/disk/:/dev/disk:ro   --publish=8080:8080  --privileged=true  --detach=true   -m 500M  --name=cadvisor  --network=opt_elastic  google/cadvisor:latest

限制内存大小
docker run --volume=/:/rootfs:ro --volume=/var/run:/var/run:rw   --volume=/sys:/sys:ro   --volume=/var/lib/docker/:/var/lib/docker:ro   --volume=/dev/disk/:/dev/disk:ro   --publish=8080:8080  --privileged=true  --detach=true   --name=cadvisor  --network=snap-net  -m 500M google/cadvisor:latest


强烈建议安装

https://grafana.com/grafana/dashboards/11190

ES Nginx Logs 模板


安装地图插件
grafana-cli plugins install raintank-worldping-app
重启grafana服务
systemctl restart grafana-server.service 

如果不出图，一般是网络问题，参考 https://cloud.tencent.com/developer/article/1623095

进入worldmap插件的安装目录备份出三个文件
1.1 grafana-worldmap-panel\src\worldmap.ts
1.2 grafana-worldmap-panel\dist\module.js
1.3 grafana-worldmap-panel\dist\module.js.map
将文件中的url进行修改.
2.1 https://cartodb-basemaps-{s}.global.ssl.fastly.net/light_all/{z}/{x}/{y}.png 修改为 http://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}.png
2.2 https://cartodb-basemaps-{s}.global.ssl.fastly.net/dark_all/{z}/{x}/{y}.png  修改为 http://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png

重启服务

**启动服务加上参数开启热加载**

nohup /usr/local/Prometheus/prometheus --config.file=/usr/local/Prometheus/prometheus.yml --web.enable-lifecycle &

热加载命令

curl -X POST http://127.0.0.1:9090/-/reload





grafana重置密码  grafana-cli admin reset-admin-password   admin@123

