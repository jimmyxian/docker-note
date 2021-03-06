### dockcer网络方式
##### bridge方式(默认)
Host IP为186.100.8.117, 容器网络为172.17.0.0/16  
下边我们看下docker所提供的四种网络：  
创建容器：（由于是默认设置，这里没指定网络--net="bridge"。另外可以看到容器内创建了eth0)  
<pre><code>
[root@localhost ~]# docker run -i -t mysql:latest /bin/bash
root@e2187aa35875:/usr/local/mysql# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
75: eth0: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:2/64 scope link
       valid_lft forever preferred_lft forever
</code></pre>
容器与Host网络是连通的：   
<pre><code>
root@e2187aa35875:/usr/local/mysql# ping 186.100.8.117
PING 186.100.8.117 (186.100.8.117): 48 data bytes
56 bytes from 186.100.8.117: icmp_seq=0 ttl=64 time=0.124 ms
</code></pre>
eth0实际上是veth pair的一端，另一端（vethb689485）连在docker0网桥上：
<pre><code>
[root@localhost ~]# ethtool -S vethb689485
NIC statistics:
     peer_ifindex: 75
[root@localhost ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.56847afe9799       no              vethb689485
</code></pre>
通过Iptables实现容器内访问外部网络：
<pre><code>
[root@localhost ~]# iptables-save |grep 172.17.0.*
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A FORWARD -d 172.17.0.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 5000 -j ACCEPT
</code></pre>

##### none方式
指定方法： --net="none"   
可以看到，这样创建出来的容器完全没有网络：   
<pre><code>
[root@localhost ~]# docker run -i -t --net="none"  mysql:latest /bin/bash
root@061364719a22:/usr/local/mysql# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
root@061364719a22:/usr/local/mysql# ping 186.100.8.117
PING 186.100.8.117 (186.100.8.117): 48 data bytes
ping: sending packet: Network is unreachable
</code></pre>
那这种方式，有什么用途呢？  
实际上nova-docker用的就是这种方式，这种方式将网络创建的责任完全交给用户。  
可以实现更加灵活复杂的网络。   
另外这种容器可以可以通过link容器实现通信。（后边详细说）     

##### host方式  
指定方法：--net="host"   
这种创建出来的容器，可以看到host上所有的网络设备。    
容器中，对这些设备（比如DUBS）有全部的访问权限。因此docker提示我们，这种方式是不安全的。     
如果在隔离良好的环境中（比如租户的虚拟机中）使用这种方式，问题不大。   

##### container复用方式  
指定方法： --net="container:name or id"   
如下例子可以看出来，两者的网络完全相同。   
<pre><code>
[root@localhost ~]# docker run -i -t   mysql:latest /bin/bash
root@02aac28b9234:/usr/local/mysql# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
77: eth0: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:3/64 scope link
       valid_lft forever preferred_lft forever
[root@localhost ~]# docker run -i -t --net="container:02aac28b9234"  mysql:latest /bin/bash
root@02aac28b9234:/usr/local/mysql# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
77: eth0: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:3/64 scope link
       valid_lft forever preferred_lft forever
</code></pre>
##### 举例（openstack nova-docker中的网络实现方式）
openstack的nova-docker插件可以向管理虚拟机一样管理容器。  
容器网络的创建方式：
首先创建--net="none"的容器，然后使用如下过程配置容器网络。（以OVS为例，也可以使用linux bridge）   
<pre><code>
#创建veth设备
ip link add name veth00 type veth peer name veth01
#将veth设备一端接入ovs网桥br-int中
ovs-vsctl -- --if-exists del-port veth00 -- add-port br-int veth00 -- set Interface veth00 external-ids:iface-id=iface_id external-ids:iface-status=active external-ids:attached-mac=00:ff:00:aa:bb:cc external-ids:vm-uuid=instance_id
#启动ovs的新加端口
ip link set veth00 up 

#配置容器的网络namespace
mkdir -p /var/run/netns
ln -sf /proc/container_pid/ns/net /var/run/netns/container_id

#将veth另一端加入容器namespace
ip link set veth01 netns container_id
#配置容器上该网络设备的mac,ip,gateway
ip netns exec container_id ip link set veth01 address mac_address
ip netns exec container_id ifconfig veth01 ip 
ip netns exec container_id ip route replace default via gateway dev veth01
</code></pre>
至此，容器与host上的虚拟网络连通。之后br-int与br-ex/br-tun连通，最终实现与业务网络的连通。

参考：  
http://segmentfault.com/blog/yexiaobai/1190000000669312
