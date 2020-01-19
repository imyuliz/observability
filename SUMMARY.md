# Table of contents

* [序言](README.md)


## 致用 PromQL<a id="promql"></a>
* [查询应用的所有实例指标, 善用group_*() on()](/promql/kubernetes_application_promql.md)

## 部署指南 <a id="deploy"></a>

* [流程分析及准备](/deploy/prepare.md)
* [相关组件](/deploy/component.md)
* [部署方式](/deploy/method/README.md)
    * [二进制部署](/deploy/method/binary.md)
    * [Docker容器部署](/deploy/method/docker.md) 
    * [Kubernetes集群内部署](/deploy/method/docker.md)

## 采集目标服务配置 <a id="server-config"></a>
* [配置文件分析](/server-config/README.md)
* [配置采集目标](/server-config/target.md)
* [Action 介绍](/server-config/action.md)
* [标签介绍](/server-config/labels/README.md)
    * [重命名标签](/server-config/labels/rename.md)
    * [删除标签](/server-config/labels/delete.md)
    * [不采集指定标签的目标](/server-config/labels/uncollect-tag.md)
    * [只采集标签的目标](/server-config/labels/only-collect-tag.md)
* [服务发现](/server-config/service-discovery/README.md)
    * [静态服务发现](/server-config/service-discovery/static.md)
    * [动态服务](/server-config/service-discovery/dynamic.md)
        * [基于文件的服务发现](/server-config/file-sd-configs.md)
        * [基于Kubernetes的服务发现](/server-config/kubernetes-sd-configs.md)
        * [基于DNS的服务发现](/server-config/dns-sd-configs.md)

## Exporter 部署
* [Node-exporter](/exporter/node-exporter.md)
## 报警服务配置 <a id="alert-config"></a>
* [报警文件分析](/alert-config/README.md)


