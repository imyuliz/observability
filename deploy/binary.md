# 二进制部署Prometheus Server

## 部署详情

prometheus server 版本:

* prometheus 2.14.0

## 下载

进入 prometheus官网的[下载页面](https://prometheus.io/download/), 在下载页面包含prometheus 支持的多种组件, 比如 xx-exporter, pushgateway 等等.

下载并使用systemctl 部署Prometheus Server

```
# 下载二进制包
wget https://github.com/prometheus/prometheus/releases/download/v2.14.0/prometheus-2.14.0.linux-amd64.tar.gz

# 解压
tar zxvf prometheus-2.14.0.linux-amd64.tar.gz
cd prometheus-2.14.0.linux-amd64
cp prometheus /usr/local/bin/
cp prometheus.yml /usr/local/bin/
```
## Systemctl 扩管服务

使用 systemctl 扩管prometheus
```
# prometheus.service
[Unit]
Description=prometheus server

[Service]
ExecStart=/usr/local/bin/prometheus --config.file=/usr/local/bin/prometheus.yml
Restart=on-failure
RestartSec=30s

[Install]
WantedBy=multi-user.target
```
## 启动 Prometheus 
使用systemctl 启动服务
```
systemctl daemon-reload
systemctl restart prometheus
```
##  检查 prometheus 启动状态
```
netstat -antp | grep 9090
# or 
ps -ef | grep prometheus
# or 
访问 http://localhost:9090/status
```




