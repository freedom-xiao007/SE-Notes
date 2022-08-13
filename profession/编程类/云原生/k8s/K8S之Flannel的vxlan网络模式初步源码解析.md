# K8S之Flannel的vxlan网络模式初步源码解析

***

## 简介

之前详细解析过Flannel的vxlan模式的网络通信原理，本篇将继续深入结合源码进行探索



## 前情提要

阅读本文需要知道flannel vxlan网络模式的网络请求路径，可以参考以前博主写的文章：[K8S探索之Service+Flannel本机及跨主机网络访问原理详解](https://juejin.cn/post/7117850911338151973)

本篇的示例场景还是之前文章中的图，如下：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9184d75062184311a0e9549b6cb90202~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

在主节点121.4.190.84，直接是可以ping通容器ip：10.244.1.8

```shell
➜  ~ ping 10.244.1.8
PING 10.244.1.8 (10.244.1.8) 56(84) bytes of data.
64 bytes from 10.244.1.8: icmp_seq=1 ttl=63 time=29.3 ms
64 bytes from 10.244.1.8: icmp_seq=2 ttl=63 time=29.3 ms
64 bytes from 10.244.1.8: icmp_seq=3 ttl=63 time=29.3 ms
```

本篇就以121.4.190.84 --> 10.244.1.8的网络请求路径开始探索下flannel的相关源码



## 源码探索

### 初始启动配置vxlan网卡

在主节点121.4.190.84，ping10.244.1.8时，由于后者不是公网ip，所以是不能直接ping的，在vxlan网络模式下，是将请求交给了vxlan网卡进行封包，封了一层后，包的请求目的地址换成了目标公网ip和vxlan的监听端口



在初次启动时，肯定得是vxlan网卡的添加了，手动配置的命令类似如下：

```shell
ip link add vxlan0 type vxlan \
   id 42 \
   dstport 4789 \
   dev enp0s8 \
   ......
```



flannel初次启动的时候肯定是要配置自己的vxlan，在机器上我们可以看到：

```sh
➜  ~ ifconfig
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.0.0  netmask 255.255.255.255  broadcast 10.244.0.0
        inet6 fe80::f4f2:b0ff:fefa:7573  prefixlen 64  scopeid 0x20<link>
        ether f6:f2:b0:fa:75:73  txqueuelen 0  (Ethernet)
        RX packets 205910  bytes 22397966 (21.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 296287  bytes 41537899 (39.6 MiB)
        TX errors 0  dropped 8 overruns 0  carrier 0  collisions 0

➜  ~ ip -d link show dev flannel.1
39: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default
    link/ether f6:f2:b0:fa:75:73 brd ff:ff:ff:ff:ff:ff promiscuity 0
    vxlan id 1 local 172.17.16.14 dev eth0 srcport 0 0 dstport 8472 nolearning ageing 300 noudpcsum noudp6zerocsumtx noudp6zerocsumrx addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535
```



在上面的查看中我们可以看flannel的vxlan网卡信息：vxlan id 1 local 172.17.16.14 dev eth0 srcport 0 0 dstport 8472 nolearning ageing 300 noudpcsum noudp6zerocsumtx noudp6zerocsumrx addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535

使用的是宿主机的ip 172.17.16.14和网卡，监听在8472端口，所以需要我们的机器在配置的时候开放8472端口，其他的参数各位有兴趣的话可以去查一查



通过相关的关键字“flannel.”在源码中搜索，在源码的目录：backend/vxlan下的vxlan.go看到一个关键函数：

```sh
func (be *VXLANBackend) RegisterNetwork(ctx context.Context, wg *sync.WaitGroup, config *subnet.Config) (backend.Network, error) {
	// Parse our configuration
	cfg := struct {
		VNI           int
		Port          int
		GBP           bool
		Learning      bool
		DirectRouting bool
	}{
		VNI: defaultVNI,
	}

	if len(config.Backend) > 0 {
		if err := json.Unmarshal(config.Backend, &cfg); err != nil {
			return nil, fmt.Errorf("error decoding VXLAN backend config: %v", err)
		}
	}
	log.Infof("VXLAN config: VNI=%d Port=%d GBP=%v Learning=%v DirectRouting=%v", cfg.VNI, cfg.Port, cfg.GBP, cfg.Learning, cfg.DirectRouting)

	var dev, v6Dev *vxlanDevice
	var err error
	if config.EnableIPv4 {
		devAttrs := vxlanDeviceAttrs{
			vni:       uint32(cfg.VNI),
			name:      fmt.Sprintf("flannel.%v", cfg.VNI),
			vtepIndex: be.extIface.Iface.Index,
			vtepAddr:  be.extIface.IfaceAddr,
			vtepPort:  cfg.Port,
			gbp:       cfg.GBP,
			learning:  cfg.Learning,
		}

		dev, err = newVXLANDevice(&devAttrs)
		if err != nil {
			return nil, err
		}
		dev.directRouting = cfg.DirectRouting
	}
	if config.EnableIPv6 {
		v6DevAttrs := vxlanDeviceAttrs{
			vni:       uint32(cfg.VNI),
			name:      fmt.Sprintf("flannel-v6.%v", cfg.VNI),
			vtepIndex: be.extIface.Iface.Index,
			vtepAddr:  be.extIface.IfaceV6Addr,
			vtepPort:  cfg.Port,
			gbp:       cfg.GBP,
			learning:  cfg.Learning,
		}
		v6Dev, err = newVXLANDevice(&v6DevAttrs)
		if err != nil {
			return nil, err
		}
		v6Dev.directRouting = cfg.DirectRouting
	}

	subnetAttrs, err := newSubnetAttrs(be.extIface.ExtAddr, be.extIface.ExtV6Addr, uint16(cfg.VNI), dev, v6Dev)
	if err != nil {
		return nil, err
	}

	lease, err := be.subnetMgr.AcquireLease(ctx, subnetAttrs)
	switch err {
	case nil:
	case context.Canceled, context.DeadlineExceeded:
		return nil, err
	default:
		return nil, fmt.Errorf("failed to acquire lease: %v", err)
	}

	// Ensure that the device has a /32 address so that no broadcast routes are created.
	// This IP is just used as a source address for host to workload traffic (so
	// the return path for the traffic has an address on the flannel network to use as the destination)
	if config.EnableIPv4 {
		if err := dev.Configure(ip.IP4Net{IP: lease.Subnet.IP, PrefixLen: 32}, config.Network); err != nil {
			return nil, fmt.Errorf("failed to configure interface %s: %w", dev.link.Attrs().Name, err)
		}
	}
	if config.EnableIPv6 {
		if err := v6Dev.ConfigureIPv6(ip.IP6Net{IP: lease.IPv6Subnet.IP, PrefixLen: 128}, config.IPv6Network); err != nil {
			return nil, fmt.Errorf("failed to configure interface %s: %w", v6Dev.link.Attrs().Name, err)
		}
	}
	return newNetwork(be.subnetMgr, be.extIface, dev, v6Dev, ip.IP4Net{}, lease)
}
```

其中的vni，port之类令人熟悉，于是我们追踪其调用链路，发现其实在系统启动时的main函数中进行调用的，符合我们的猜测，是在flannel启动时进行配置

接着反过来

其中还发现了，linux和Windows有不同的实现文件，而不同系统好像是通过下面的源码中的关键字生效的,又学到了一手

```go
//go:build !windows
// +build !windows
```

当然其中还有很多的细节，留待后面探索，本篇先初步关键请求链路熟悉下源码，知道了核心路径后，出问题想查看的话，能提供思路上的帮助



### 路由和fdb的自动配置

在上面配置好vxlan网卡后，我们的请求经过大致如下：

1. 请求10.244.1.8，转到本机的vxlan网卡flannel.1进行处理
2. 将目标公网ip：106.55.227.160作为封包后的目标ip



上面两步都需要进行配置，首先1是要配置本机的路由，让10.244.1.8的请求转到网卡flannel.1进行处理，我们查看机器上的路由，确实是有，如下：

```sh
➜  ~ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.244.1.0      10.244.1.0      255.255.255.0   UG    0      0        0 flannel.1
10.244.4.0      10.244.4.0      255.255.255.0   UG    0      0        0 flannel.1
```

其中就吧10.244.1.0/24 这个网段都交给了网卡flannel.1进行处理

路由配置这一步，得是flannel进行处理，如新增了一台机器（目前试验下来，一台机器对应一个子网），需要添加相关的路由



第二步也很关键，请求的是10.244.1.8，vxlan怎么知道目标地址是公网：106.55.227.160，这个是通过fdb进行实现的（vxlan这个功能都是操作系统已经有的，如果看到这迷糊，需要仔细再阅读下vxlan的原理介绍，可参考书籍：《Kubenetes网络权威指南：基础、原理与实践》），我们参考flannel的fdb可以看到：

```sh
➜  ~ bridge fdb show dev flannel.1
da:a9:48:cb:15:3b dst 106.55.227.160 self permanent
c6:8a:3b:81:2f:d1 dst 121.37.246.218 self permanent

# 添加命令
bridge fdb append da:a9:48:cb:15:3b dev flannel.0 dst 106.55.227.160
```

上面的fdb就是目标机器人Mac地址和公网ip的关系，如da:a9:48:cb:15:3b对应的就是106.55.227.160，而106.55.227.160是机器106.55.227.160的vxlan网卡:

```sh
➜  ~ ifconfig
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.1.0  netmask 255.255.255.255  broadcast 10.244.1.0
        inet6 fe80::d8a9:48ff:fecb:153b  prefixlen 64  scopeid 0x20<link>
        ether da:a9:48:cb:15:3b  txqueuelen 0  (Ethernet)
```

那还有一个疑问，我请求的10.244.1.8，你怎么要请求发到106.55.227.160的vxlan网卡上，这个就是arp起的作用了，我们查看arp如下：

```sh
➜  ~ ip neigh
10.244.1.0 dev flannel.1 lladdr da:a9:48:cb:15:3b PERMANENT

# 添加命令
ip neigh add 10.244.1.0 lladdr da:a9:48:cb:15:3b flannel.1
```

大致的意思就是告诉这个ip的Mac地址是多少



通过上面的描述，我们知道了更详情的请求处理路径如下：

1. ping 10.244.1.8
2. 通过路由表配置，得到目标网络为10.224.1.0的都交给vxlan网卡处理
3. 通过arp配置：10.224.1.0在arp中得到其对应目标机器Mac地址
4. 通过fdb配置：得到Mac地址对应的ip地址

这样，请求就成功的抵达对面了



下面我们看下flannel中的相关源码

首先我们在运行日志中看到一个监听类型启动成功的日志：watching for new subnet leases

```sh
➜  ~ kubectl logs kube-flannel-ds-vwn7x -n kube-system
I0707 04:00:26.313804       1 main.go:533] Using interface with name eth0 and address 172.17.16.14
I0707 04:00:26.313867       1 main.go:546] Using 121.4.190.84 as external address
W0707 04:00:26.313888       1 client_config.go:608] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0707 04:00:26.419108       1 kube.go:116] Waiting 10m0s for node controller to sync
I0707 04:00:26.419153       1 kube.go:299] Starting kube subnet manager
I0707 04:00:27.419250       1 kube.go:123] Node controller sync successful
I0707 04:00:27.419274       1 main.go:254] Created subnet manager: Kubernetes Subnet Manager - crio-master
I0707 04:00:27.419279       1 main.go:257] Installing signal handlers
I0707 04:00:27.419343       1 main.go:392] Found network config - Backend type: vxlan
I0707 04:00:27.419401       1 vxlan.go:123] VXLAN config: VNI=1 Port=0 GBP=false Learning=false DirectRouting=false
I0707 04:00:27.439715       1 main.go:307] Setting up masking rules
I0707 04:00:27.520123       1 main.go:315] Changing default FORWARD chain policy to ACCEPT
I0707 04:00:27.520207       1 main.go:323] Wrote subnet file to /run/flannel/subnet.env
I0707 04:00:27.520223       1 main.go:327] Running backend.
I0707 04:00:27.520233       1 main.go:345] Waiting for all goroutines to exit
I0707 04:00:27.520258       1 vxlan_network.go:59] watching for new subnet leases
```

直接在源码中找到：

```go
# 注意是在vxlan_network.go文件中
func (nw *network) Run(ctx context.Context) {
	wg := sync.WaitGroup{}

	log.V(0).Info("watching for new subnet leases")
	events := make(chan []subnet.Event)
	wg.Add(1)
	go func() {
		subnet.WatchLeases(ctx, nw.subnetMgr, nw.SubnetLease, events)
		log.V(1).Info("WatchLeases exited")
		wg.Done()
	}()

	defer wg.Wait()

	for {
		evtBatch, ok := <-events
		if !ok {
			log.Infof("evts chan closed")
			return
		}
		nw.handleSubnetEvents(evtBatch)
	}
}
```

大意是启动了一个协程去获取容器添加（通过之前的分析，机器增加和容器增加应该要维护路由/arp/fdb表）引起的事件，在当前线程中进行事件的相关处理

使用channel进行同步通信实现，一个写一个读

subnet.WatchLeases(ctx, n.SM, n.SubnetLease, evts)，在查看的过程中还有两种不同的实现，分别是local和kubenetes，猜测前者是不依赖与kubenetes的，单独部署的情况下启动的，这部分就不分析，kubenetes的源码还没有开始搞，留待日后分析

看看事件处理的函数，下面是处理网络添加相关：

```sh
func (nw *network) handleSubnetEvents(batch []subnet.Event) {
	for _, event := range batch {
		sn := event.Lease.Subnet
		v6Sn := event.Lease.IPv6Subnet
		attrs := event.Lease.Attrs
		if attrs.BackendType != "vxlan" {
			log.Warningf("ignoring non-vxlan v4Subnet(%s) v6Subnet(%s): type=%v", sn, v6Sn, attrs.BackendType)
			continue
		}

		var (
			vxlanAttrs, v6VxlanAttrs           vxlanLeaseAttrs
			directRoutingOK, v6DirectRoutingOK bool
			directRoute, v6DirectRoute         netlink.Route
			vxlanRoute, v6VxlanRoute           netlink.Route
		)

        # 路由添加相关的
		if event.Lease.EnableIPv4 && nw.dev != nil {
			if err := json.Unmarshal(attrs.BackendData, &vxlanAttrs); err != nil {
				log.Error("error decoding subnet lease JSON: ", err)
				continue
			}

			// This route is used when traffic should be vxlan encapsulated
			vxlanRoute = netlink.Route{
				LinkIndex: nw.dev.link.Attrs().Index,
				Scope:     netlink.SCOPE_UNIVERSE,
				Dst:       sn.ToIPNet(),
				Gw:        sn.IP.ToIP(),
			}
			vxlanRoute.SetFlag(syscall.RTNH_F_ONLINK)

			// directRouting is where the remote host is on the same subnet so vxlan isn't required.
			directRoute = netlink.Route{
				Dst: sn.ToIPNet(),
				Gw:  attrs.PublicIP.ToIP(),
			}
			if nw.dev.directRouting {
				if dr, err := ip.DirectRouting(attrs.PublicIP.ToIP()); err != nil {
					log.Error(err)
				} else {
					directRoutingOK = dr
				}
			}
		}
        ······
        # arp和fdb添加相关的
		switch event.Type {
		case subnet.EventAdded:
			if event.Lease.EnableIPv4 {
				if directRoutingOK {
					log.V(2).Infof("Adding direct route to subnet: %s PublicIP: %s", sn, attrs.PublicIP)

					if err := netlink.RouteReplace(&directRoute); err != nil {
						log.Errorf("Error adding route to %v via %v: %v", sn, attrs.PublicIP, err)
						continue
					}
				} else {
					log.V(2).Infof("adding subnet: %s PublicIP: %s VtepMAC: %s", sn, attrs.PublicIP, net.HardwareAddr(vxlanAttrs.VtepMAC))
					if err := nw.dev.AddARP(neighbor{IP: sn.IP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)}); err != nil {
						log.Error("AddARP failed: ", err)
						continue
					}

					if err := nw.dev.AddFDB(neighbor{IP: attrs.PublicIP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)}); err != nil {
						log.Error("AddFDB failed: ", err)

						// Try to clean up the ARP entry then continue
						if err := nw.dev.DelARP(neighbor{IP: event.Lease.Subnet.IP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)}); err != nil {
							log.Error("DelARP failed: ", err)
						}

						continue
					}

					// Set the route - the kernel would ARP for the Gw IP address if it hadn't already been set above so make sure
					// this is done last.
					if err := netlink.RouteReplace(&vxlanRoute); err != nil {
						log.Errorf("failed to add vxlanRoute (%s -> %s): %v", vxlanRoute.Dst, vxlanRoute.Gw, err)

						// Try to clean up both the ARP and FDB entries then continue
						if err := nw.dev.DelARP(neighbor{IP: event.Lease.Subnet.IP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)}); err != nil {
							log.Error("DelARP failed: ", err)
						}

						if err := nw.dev.DelFDB(neighbor{IP: event.Lease.Attrs.PublicIP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)}); err != nil {
							log.Error("DelFDB failed: ", err)
						}

						continue
					}
				}
			}
			······
		case subnet.EventRemoved:
			······
		default:
			log.Error("internal error: unknown event type: ", int(event.Type))
		}
	}
}

```

如上所示，在源码中也找到了route、arp、fdb相关的配置，猜测得到了印证



## 总结

本篇文章中通过分析ip请求的路径，结合如何手动配置，然后去找对应的源码进行印证，大致找到了flannel源码的vxlan核心代码，但没有具体分析，这个感觉还是需要时间和精力的，所以本篇就初步探索下，细节日后分析



总结来说核心就是基于操作系统的vxlan，使用程序代替手动配置route、arp、fdb表，实现了跨主机的容器通信：

1. ping 10.244.1.8
2. 通过路由表配置，得到目标网络为10.224.1.0的都交给vxlan网卡处理
3. 通过arp配置：10.224.1.0在arp中得到其对应目标机器Mac地址
4. 通过fdb配置：得到Mac地址对应的ip地址