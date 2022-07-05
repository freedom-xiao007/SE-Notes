# K8S 应用动态访问
***



```sh
➜ curl http://10.1.167.223:9000/app/versionCheck\?version\=1
{"data":{"downloadUrl":null,"updateMsg":null,"latest":true},"code":200,"msg":null}# 

"interface=eth0"
```

## cailco 模式重新配置
```sh
kubectl delete ippools --all
./calicoctl apply -f ipip.yaml

route add -net 192.168.3.197 netmask 255.255.255.0 gw 10.10.10.1
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

### Readiness probe failed: calico/node is not ready: BIRD is not ready: Error querying BIRD: unable to connect to BIRDv4 socket: dial unix /var/run/calico/bird.ctl: connect: connection refused

- [kk添加node节点，calico组件启动不了](https://kubesphere.com.cn/forum/d/3129-kknodecalico)
- [安装calico一个组件0/1 unable to connect to BIRDv4 socket: dial unix /var/run/bird/bird.ctl: connect: no su](https://blog.csdn.net/MrFDd/article/details/123358476)
- [k8s安装calico的时候报：BIRD is not ready](https://blog.csdn.net/xing_S/article/details/123630179?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-123630179-blog-123358476.pc_relevant_aa2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-123630179-blog-123358476.pc_relevant_aa2&utm_relevant_index=2)

## 参考链接

- [使用 Service 暴露你的应用](https://kubernetes.io/zh-cn/docs/tutorials/kubernetes-basics/expose/expose-intro/)
- [Kubernetes系列之五：使用yaml文件创建service向外暴露服务](https://blog.csdn.net/wucong60/article/details/81699196)
- [Kubernetes对象之Service](https://www.cnblogs.com/tylerzhou/p/10989881.html)
- [linux ip 转发设置 ip_forward、ip_forward与路由转发](https://blog.csdn.net/li_101357/article/details/78416813)
- [calico Overlay networking](https://projectcalico.docs.tigera.io/networking/vxlan-ipip#configure-vxlan-encapsulation-for-all-inter-workload-traffic)
- [k8s中正确删除一个pod](https://www.cnblogs.com/effortsing/p/10496547.html)
- [强制删除POD](https://www.jianshu.com/p/fe7473e43d76)
- [tcpdump详解和定位解决问题实例分析](https://blog.csdn.net/wj31932/article/details/106570542)