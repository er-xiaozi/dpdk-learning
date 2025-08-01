## 安装

NDNDPDK_MK_RELEASE=1 make
cd mk/
sudo ./install.sh

## NDN-DPDK

### 1、以太网Face  转发器的激活与使用

创建步骤

* 在以太网适配器上创建以太网端口

  ```
  ndndpdk-ctrl create-eth-port #创建以太网端口 返回JSON 包含太网适配器的 DPDK 设备名称和本地 MAC 地址
  ndndpdk-ctrl create-ether-faceid #该命令将创建一个以太网面。 它返回一个 JSON 对象，该对象包含一个属性，其值是人脸的不透明标识符。
  ```

  

* 在以太网端口上创建一个基于以太网的Face

  ```
  A $ ndndpdk-ctrl create-eth-port --pci 04:00.0 --mtu 9000
  {"id":"1276dc31","macAddr":"02:00:00:00:00:01","name":"0000:04:00.0","numaSocket":1}
  # to use '--mtu 9000', .mempool.DIRECT.dataroom in activation parameters should be at least 9146
  
  B $ ndndpdk-ctrl create-eth-port --pci 06:00.0 --mtu 9000
  {"id":"a6a35a10","macAddr":"02:00:00:00:00:02","name":"0000:06:00.0","numaSocket":1}
  
  A $ ndndpdk-ctrl create-ether-face --local 02:00:00:00:00:01 --remote 02:00:00:00:00:02
  {"id":"286d21ff"}
  
  B $ ndndpdk-ctrl create-ether-face --local 02:00:00:00:00:02 --remote 02:00:00:00:00:01
  {"id":"31bfaaa9"}
  ```

  

### 启用 IOMMU 绑定PCI驱动程序

修改/etc/default/grub, 调整GRUB_CMDLINE_LINUX内容

GRUB_CMDLINE_LINUX="crashkernel=auto rhgb quiet intel_iommu=on iommu=pt"

grub-mkconfig -o /boot/efi/EFI/ubuntu/grub.cfg

dmesg | grep -e DMAR -e IOMMU

加载驱动

sudo modprobe vfio-pci

查看网卡PCI地址

lspci

sudo ifconfig ens37 down

你提到的两个 Realtek 千兆以太网控制器的 PCI 地址分别是：

- **02:00.0**
- **04:00.0**

加载 `vfio-pci` 模块

sudo modprobe vfio-pci

sudo dpdk-devbind.py --bind=vfio-pci 03:00.0

如果系统上没有可用的 IOMMU，则必须通过 VFIO-pci 启用不安全模式进行绑定，VFIO 仍然可以使用，但必须加载额外的模块参数：

```
modprobe vfio enable_unsafe_noiommu_mode=1
```

或者，也可以在已经加载的 kernel 模块中启用此选项：

```
echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
```

```
sudo dpdk-devbind.py -b uio_pci_generic 04:00.0
```

### 配置 Hugepages

echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

### 启动

```
sudo ndndpdk-ctrl systemd start ndndpdk-ctrl ndndpdk-ctrl systemd logs -f
```

sudo ndndpdk-ctrl systemd start

sudo ndndpdk-ctrl system logs -f

测试连接

sudo ndndpdk-godemo pingclient –name /example/P.

/home/ujs410/Desktop/



流量生成器激活参数

ndndpdk-ctrl activate-trafficgen < /home/lwj/Desktop/trafficgen.schema.json

ndndpdk-ctrl activate-trafficgen < /home/m/桌面/trafficgen.schema.json

sudo ndndpdk-ctrl start-trafficgen < /home/m/桌面/gen.schema.json



ndndpdk-ctrl create-eth-port --pci 04:00.0 --mtu 1500

`RTL8111/8168/8411` 网卡可能不完全支持 DPDK

驱动可能不兼容DPDK  尝试使用 `igb_uio` 驱动

enp2s0

02:05.0

02:01.0 

trafficgen.schema.json

```
{
  "eal": {
    "cores": [ 0, 1, 2, 3 ],
    "memChannels": 4,
    "disablePCI": false,
    "filePrefix": "ndn",
	"iovaMode": "PA"
  },
  "lcoreAlloc": {
    "HRLOG": [ 2 ],
    "PDUMP": [ 3 ]
  },
  "mempool": {
    "DATA": {
      "capacity": 8192,
      "dataroom": 2176
    },
    "INTEREST": {
      "capacity": 8192,
      "dataroom": 2176
    }
  },
  "socketFace": {
    "rxConns": {
      "ringCapacity": 4096
    },
    "txSyscall": {
      "disabled": false
    }
  }
}

```



```
# 启用 IOMMU
编辑 GRUB 配置文件：
sudo nano /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash iommu=pt intel_iommu=on iova_mode=pa"
# 修改/etc/default/grub, 调整GRUB_CMDLINE_LINUX内容
GRUB_CMDLINE_LINUX="crashkernel=auto rhgb quiet intel_iommu=on iommu=pt"
grub-mkconfig -o /boot/efi/EFI/ubuntu/grub.cfg
# 设置巨页
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
# 启动主进程
ndndpdk-svc
# 查网卡
lspci | grep Ethernet
# down 网卡
sudo ifconfig ens160 down
# 绑网卡
sudo modprobe vfio-pci
sudo dpdk-devbind.py -b vfio-pci 03:00.0

# 如果报错
如果系统上没有可用的 IOMMU，则必须通过 VFIO-pci 启用不安全模式进行绑定，VFIO 仍然可以使用，但必须加载额外的模块参数：
modprobe vfio enable_unsafe_noiommu_mode=1
或者，也可以在已经加载的 kernel 模块中启用此选项：
echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode

# 激活流量生成器  执行trafficgen.schema.json
ndndpdk-ctrl activate-trafficgen < /home/lwj/Desktop/trafficgen.schema.json
# 绑定face 
ndndpdk-ctrl create-eth-port --pci 03:00.0 --mtu 1500
## 如果绑定不成功需要先再跑一次激活流量生成器 再绑定流量生成器
## 可能有用export RTE_EAL_FLAGS="--iova-mode=pa"
成功显示
{"id":"N89DLCNIAIDGM","isDown":true,"macAddr":"00:0c:29:a1:cb:21","mtu":1500,"name":"0000:03:00.0","numaSocket":null}

# 启动流量生成器
sudo ndndpdk-ctrl start-trafficgen < /home/lwj/Desktop/gen.schema.json

# 关闭主进程
sudo killall ndndpdk-svc

```

* 转发器 

ndndpdk-ctrl activate-forwarder < /home/lwj/Desktop/forwarder.schema.json

```
ndndpdk-ctrl activate-forwarder < /home/ujs410/Desktop/fw-args.json
```

```\
A $ ndndpdk-ctrl create-eth-port --pci 04:00.0 --mtu 1500
{"id":"1276dc31","macAddr":"02:00:00:00:00:01","name":"0000:04:00.0","numaSocket":1}
# to use '--mtu 9000', .mempool.DIRECT.dataroom in activation parameters should be at least 9146

B $ ndndpdk-ctrl create-eth-port --pci 06:00.0 --mtu 1500
{"id":"a6a35a10","macAddr":"02:00:00:00:00:02","name":"0000:06:00.0","numaSocket":1}

A $ ndndpdk-ctrl create-ether-face --local 00:0c:29:a1:cb:21 --remote 00:0c:29:57:30:2a
{"id":"286d21ff"}

B $ ndndpdk-ctrl create-ether-face --local 00:0c:29:57:30:2a --remote 00:0c:29:a1:cb:21
{"id":"31bfaaa9"}
```

```
#插入FIB
A $ ndndpdk-ctrl insert-fib --name /example/P --nh 286d21ff
{"id":"5aa50b21"}
```

启动

```
B $ ndndpdk-godemo pingserver --name /example/P
2022/05/05 14:54:17 uplink opened, state is down
2022/05/05 14:54:17 uplink state changes to up
2022/05/05 14:54:18 /8=example/8=P/8=0E0344249FD27C3A[F]
2022/05/05 14:54:18 /8=example/8=P/8=0E0344249FD27C3B[F]
2022/05/05 14:54:19 /8=example/8=P/8=0E0344249FD27C3C[F]
2022/05/05 14:54:19 /8=example/8=P/8=0E0344249FD27C3D[F]
2022/05/05 14:54:19 /8=example/8=P/8=0E0344249FD27C3E[F]
2022/05/05 14:54:19 /8=example/8=P/8=0E0344249FD27C3F[F]
2022/05/05 14:54:40 uplink state changes to down
2022/05/05 14:54:40 uplink closed, error is <nil>

A $ ndndpdk-godemo pingclient --name /example/P
2022/05/05 14:54:18 uplink opened, state is down
2022/05/05 14:54:18 uplink state changes to up
2022/05/05 14:54:18 100.00% D 0E0344249FD27C3A   1294us
2022/05/05 14:54:18 100.00% D 0E0344249FD27C3B   1685us
2022/05/05 14:54:19 100.00% D 0E0344249FD27C3C    710us
2022/05/05 14:54:19 100.00% D 0E0344249FD27C3D    643us
2022/05/05 14:54:19 100.00% D 0E0344249FD27C3E   1182us
2022/05/05 14:54:19 100.00% D 0E0344249FD27C3F   1975us
2022/05/05 14:54:19 uplink state changes to down
2022/05/05 14:54:19 uplink closed, error is <nil>
```



forwarder.schema.json

```
{
  "eal": {
    "cores": [0, 1, 2, 3, 5, 6, 7],
    "memChannels": 4,
    "disablePCI": false,
    "filePrefix": "ndn-forwarder",
    "iovaMode": "PA"
  },
  "lcoreAlloc": {
    "HRLOG": [1],
    "FWD": [1],
    "RX": [1],
    "TX": [0]
  },
  "fib": {
    "capacity": 1024,
    "nBuckets": 64,
    "startDepth": 1
  },
  "mempool": {
    "DIRECT": {
      "capacity": 8192,
      "dataroom": 2176
    },
    "INDIRECT": {
      "capacity": 8192,
      "dataroom": 2176
    }
  },
  "socketFace": {
    "rxConns": {
      "ringCapacity": 4096
    },
    "txSyscall": {
      "disabled": false
    }
  }
}

```

```
# cup核心不够时采用下面的
{
  "eal": {
    "coresPerNuma": {
      "0": 2
    },
    "lcoresPerNuma": {
      "0": 6
    },
    "lcoreMain": 3,
    "filePrefix": "ndn-forwarder",
    "iovaMode": "PA"
  },
  "lcoreAlloc": {
    "RX": { "0": 1 },
    "TX": { "0": 1 },
    "FWD": { "0": 2 },
    "CRYPTO": { "0": 0 }
  },
  "mempool": {
    "DIRECT": {
      "capacity": 24287,
      "dataroom": 9146
    },
    "INDIRECT": {
      "capacity": 24287
    }
  },
  "fib": {
    "capacity": 4095,
    "startDepth": 8
  },
  "pcct": {
    "pcctCapacity": 65535,
    "csMemoryCapacity": 20000,
    "csIndirectCapacity": 20000
  }
}
```

```
ndndpdk-ctrl create-eth-port --pci 03:00.0 --mtu 1500
```

ndndpdk-ctrl list-face

ndndpdk-ctrl get-face --id P6EHH1IMQQCA5DLI --cnt | jq .counters

```
ndndpdk-ctrl create-eth-port --pci 02:00.0 --mtu 1500
```

sudo ifconfig enp2s0 down

ndndpdk-ctrl create-eth-port --netif enp4s0 --xdp --mtu 1500

```
ndndpdk-ctrl create-eth-port --netif enp4s0 --mtu 1500
```

**手动构建和安装libxdp**：

git clone https://github.com/xdp-project/xdp-tools.git
cd xdp-tools

ndndpdk-ctrl create-ether-face --local 00:0c:29:a1:cb:21 --remote 00:0c:29:77:01:52



## NDN基础实验

快速开始

cd /home/lwj/Desktop/ndn-dpdk

sudo killall ndndpdk-svc

NDNDPDK_MK_RELEASE=1 make
cd mk/
sudo ./install.sh

```
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
ndndpdk-svc
sudo ifconfig ens160 down
sudo modprobe vfio-pci
modprobe vfio enable_unsafe_noiommu_mode=1
echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
sudo dpdk-devbind.py -b vfio-pci 03:00.0
ndndpdk-ctrl activate-forwarder < /home/lwj/Desktop/forwarder.schema.json
ndndpdk-ctrl create-eth-port --pci 03:00.0 --mtu 1500
```

主机A

```
ndndpdk-ctrl activate-forwarder < /home/lwj/Desktop/forwarder.schema.json
ndndpdk-ctrl create-eth-port --pci 03:00.0 --mtu 1500
ndndpdk-ctrl create-ether-face --local 00:0c:29:a1:cb:21 --remote 00:0c:29:57:30:2a
```

主机B

```
ndndpdk-ctrl create-ether-face --local 00:0c:29:57:30:2a --remote 00:0c:29:a1:cb:21
```

主机A插入FIB

```
ndndpdk-ctrl insert-fib --name /example/P --nh 3O8JGALFV96LPVG
```

ping

```
主机 A $ ndndpdk-godemo pingclient --name /example/P
主机 B $ ndndpdk-godemo pingserver --name /example/P
```

计数器

```
ndndpdk-ctrl list-face
ndndpdk-ctrl get-face --id PFLDAG3BHQPBB66T --cnt | jq .counters
```

fmt.Printf("Interest: %+v\n", pkt.Interest)

​      fmt.Printf("Pkt: %+v\n", pkt)

```
ndndpdk-svc
ndndpdk-ctrl activate-forwarder < /home/lwj/Desktop/forwarder.schema.json
ndndpdk-ctrl create-eth-port --pci 03:00.0 --mtu 1500
ndndpdk-ctrl create-ether-face --local 00:0c:29:57:30:2a --remote 00:0c:29:a1:cb:21
ndndpdk-godemo pingserver --name /example/P
```

