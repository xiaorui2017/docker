## 多主机网络

#### 网络术语的概念

**二层交换技术：**工作在OSI七层网络模型的第二层（数据链路层），通过MAC地址进行帧的转发

**三层交换技术：**也称为IP交换技术，工作在OSI七层网络模型的第三层（网络层），通过IP地址进行包转发。它解决了局域网中网段划分后，网段中子网必须依赖路由器进行管理的局面。结合了二层的数据块。

**网桥（Bridge）：**工作在OSI七层网络模型的第二层，根据MAC地地址转发，类似于二层交换机。网桥决定了接收的数据包是转发给同一个局域网内主机还是别的网络。

**VLAN（虚拟局域网）：**在物理网络（通常在路由器接口）基础上建立一个或者多个逻辑子网，将一个大的广播域切分若干个小的广播域。一个VLAN就是一个广播域，VLAN之间通信通过三层路由器来完成。

**Overlay Network：**覆盖网络，在基础网络上叠加一种虚拟网络技术模式，该网络中的主机通过虚拟链路连接起来。

- Overlay网络有以下三种实现方式：

  - **VXLAN**（虚拟可扩展局域网），通过将物理服务器或虚拟机发出的数据包封装到UDP中，并使用武力网络的IP/MAC作为外层报文头进行封装，然后再IP网络上传输，到达目的地后又隧道端点解封装并将数据发送给目标物理服务器或虚拟机，扩展了大规模虚拟机网络通信。

    - 由于**VLAN Header**头部限制长度是12bit，导致只能分配4095个VLAN，也就是4095网段。在大规模虚拟网络，VXLAN标准定义Header先绘制长度24bit，可以支持1600万个VLAN。

    - **VXLAN技术核心组成：**

      **NVE（网络虚拟端点）**：实现网络虚拟化功能。报文经过NVE封装转换后，NVE间就可以基于三层基础网络建立二层虚拟化网络

      **VTEP（VXLAN隧道端点）**：封装在NVE中，用于VXLAN报文的封装和解封装。

      **VNI（VXLAN网络标识ID）**：类似于VLAN ID，用于区分VXLAN段，不同的CVXLAN段不能直接二层网络通信。

      ![](.\img\2018-10-15_151446.png)

  - **NVGRE（使用GRE虚拟网络）：**没有采用标准的传输协议（TCP/UDP），而是借助路由封装协议（GRE）。采用24bit标识二层网络分段，与VXLAN一样可以支持1600万个虚拟网络。

  - **STT（无状态传输隧道）：**模拟TCP数据格式进行封装，改造了TCP传输机制，不维护TCP状态信息。



#### 容器跨主机通信主流方案

##### Docker主机之间容器通信解决方案

- 桥接宿主机网络
- 端口映射
- docker网络驱动
  - Overlay：基于VXLAN封装实现Docker原生overlay网络
  - Macvlan:Docker主机网卡逻辑分出多个子接口，每个子接口表示一个VLAN。容器接口直接连接Dockers主机网卡接口，通过路由策略转发到另一台Docker主机
- 第三网络项目
  - 隧道方案
    - Flannel：支持UDP和VXLAN封装传输方式
    - Weave：支持UDP（sleeve模式）和VXLAN（优先fastdp模式）
    - OpenvSwitch：支持VXLAN和GRE协议
  - 路由方案
    - Calico：支持BGP协议和IPIP隧道。每台宿主机作为虚拟路由，通过BGP协议实现不同主机容器间通信



#### Docker原生Overlay网络部署

需要满足以下任意条件：

- Docker运行在Swarm模式
- 使用键值存储的Dockers主机集群

这里演示第二种，需要满足以下条件：

- 集群中主机连接到键值存储，Docker支持Consul、Etcd和Zookeeper；
- 集群中主机运行一个Dockers守护进程；
- 集群中主机必须具有唯一的主机名，因为键值存储使用主机名来标识集群成员；
- 集群中Linux主机内核版本3.12+，支持VXLAN数据包处理，否则可能无法通信。

```shell
节点1/键值存储：192.168.1.198
节点2：192.168.1.199
1. 下载Consul 二进制包并启动
# wget https://releases.hashicorp.com/consul/0.9.2/consul_0.9.2_linux_amd64.zip
# unzip consul_0.9.2_linux_amd64.zip
# mv consul /usr/bin/consul && chmod +x /usr/bin/consul
# nohup consul agent -server -bootstrap -ui -data-dir /var/lib/consul -client=192.168.1.198 -bind=192.168.1.198
&>/var/log/consul.log &
2 2. . 节点配置 Docker 守护进程连接 Consul
# vi /lib/systemd/system/docker.service
[Service]
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store consul:// 192.168.1.198:8500
--cluster-advertise 192.168.1.198:2375
# systemctl restart docker
3. 创建 overlay 网络
# docker network create -d overlay multi_host
4 4. . 测试互通
# docker run -itd --net=multi_host busybox
```

![](.\img\2018-10-15_202731.png)

#### Docker Macvlan Network 网络部署

##### **MacvlanBridge模式：**

##### 1.创建macvlan网络

```shell
dockernetwork create-d macvlan--subnet=172.100.1.0/24--gateway=172.100.1.1-oparent=eth0 macvlan_net
```

**2.测试互通**

```shell
macvlan-01# dockerrun-it--net macvlan_net--ip=172.100.1.10 busybox
macvlan-02# dockerrun-it--net macvlan_net--ip=172.100.1.11 busybox ping 172.100.1.10
```

##### MacvlanVLAN Bridge模式：

**1.创建一个VLAN，VLAN ID 50**

```shell
ip link add link eth0 name eth0.50 type vlan id 50
```

**2.创建Macvlan网络**

```shell
docker network create -d macvlan --subnet=172.18.50.0/24 --gateway=172.18.50.1 -o parent=eth0.50 macvlan_net50
```

**3.测试互通**

```shell
macvlan-01# docker run -it --net macvlan_net50 --ip=172.18.50.10 busybox
macvlan-02# docker run -it --net macvlan_net50 --ip=172.18.50.11 busybox ping 172.18.50.10
```



