# efk
efk收集日志
流程是这个样子，还有什么细节需要注意的么。
1. 创建namespace : logging ,后续所有资源在logging namespace下部署
2. StatefulSet 创建es数据库
3. 创建es对应的无头服务Service
4.创建 ServiceAccount fluentd 用于fluentd pod yaml的配置
5.ClusterRoleBinding 给创建的fluentd 绑定权限
6.daemonset创建 fluentd搜集 镜像： fluentd-kubernetes-daemonset
7.部署 kibana Pod
8.部署 kibana Service
9.部署ingress 访问 kibana
