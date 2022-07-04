# K8S 应用动态访问
***



```sh
➜ curl http://10.1.169.114:32143/app/versionCheck\?version\=1
{"data":{"downloadUrl":null,"updateMsg":null,"latest":true},"code":200,"msg":null}# 
```



## 问题记录
### 重新安装CNI插件后，node节点一直notready
查看kubelet日志，一直报错如下：

```sh
m runtime service failed" err="rpc error: code = Unknown desc = failed to destroy network for sandbox \"f8b273d0b213af9a143f5729da56365efaf8fb322a1299ce2cfb8eef2cbd03c2\": cni plugin not initialized" podSandboxID="f8b273d0b213af9a14
dbox before removing" err="rpc error: code = Unknown desc = failed to destroy network for sandbox \"f8b273d0b213af9a143f5729da56365efaf8fb322a1299ce2cfb8eef2cbd03c2\": cni plugin not initialized" sandboxID="f8b273d0b213af9a143f5729d
m runtime service failed" err="rpc error: code = Unknown desc = failed to destroy network for sandbox \"e6a73ececa384f6946f9d764a093f639901fc647d84cd32f0175505a72b50f47\": cni plugin not initialized" podSandboxID="e6a73ececa384f6946
dbox before removing" err="rpc error: code = Unknown desc = failed to destroy network for sandbox \"e6a73ececa384f6946f9d764a093f639901fc647d84cd32f0175505a72b50f47\": cni plugin not initialized" sandboxID="e6a73ececa384f6946f9d764a
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
```

解决办法，重新启动下contained：

```sh
systemctl daemon-reload
systemctl restart containerd
```

## 参考链接

- [使用 Service 暴露你的应用](https://kubernetes.io/zh-cn/docs/tutorials/kubernetes-basics/expose/expose-intro/)
- [Kubernetes系列之五：使用yaml文件创建service向外暴露服务](https://blog.csdn.net/wucong60/article/details/81699196)
- [Kubernetes对象之Service](https://www.cnblogs.com/tylerzhou/p/10989881.html)
- [linux ip 转发设置 ip_forward、ip_forward与路由转发](https://blog.csdn.net/li_101357/article/details/78416813)
- [calico Overlay networking](https://projectcalico.docs.tigera.io/networking/vxlan-ipip#configure-vxlan-encapsulation-for-all-inter-workload-traffic)