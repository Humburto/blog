+++
title = '配置一个全局代理的虚拟机'
date = 2025-02-10T14:51:32Z
draft = false
+++

## Introduction
在本地开发中经常碰到各种特色网络问题，借助tun2socks配置一个全局代理的虚拟机作为开发环境能一劳永逸的解决这一问题

## 配置全局代理
1. 下载解压[tun2socks](https://github.com/xjasonlyu/tun2socks)
2. 创建tun设备，配置ip，配置路由，可以用如下python脚本完成：
```python
# setup.py
import os
import subprocess

def run(cmd, check=True):
    print(f"[RUN] {cmd}")
    f = subprocess.check_call if check else subprocess.call
    f(cmd, shell=True)

def main():
    dev = 'tun0'
    if not os.path.exists(f'/sys/class/net/{dev}'):
        run('ip tuntap add mode tun dev tun0')
        run(f'ip addr add 198.18.0.1/15 dev {dev}')
    run(f'sudo ip link set dev {dev} up')
    run(f'ip route add default via 198.18.0.1 dev {dev} metric 1')

if __name__ == '__main__':
    main()
```
3. 编写清理脚本，在tun2socks停止后恢复路由
```bash
# cleanup.sh
sudo sudo ip route delete default via 198.18.0.1 dev tun0 metric 1
```
4. 启动tun2socks，可以使用systemd配置成一个服务方便管理和开机自启，参考如下.service配置文件
```ini
[Unit]
Description=tun2socks
After=network.target network-online.target nss-lookup.target

[Service]
Type=simple  
StandardError=journal     
StandardOutput=journal
ExecStartPre=/usr/bin/python3 /path/to/setup.py
ExecStart=/path/to/tun2socks-linux-amd64 -device tun0 -proxy socks5://<proxy_ip>:<port> -interface <物理网卡>
ExecStop=/usr/bin/bash /path/to/cleanup.sh
WorkingDirectory=/path/to/workdir
LimitNOFILE=51200
Restart=on-failure
RestartSec=1s
                                                                                                        [Install]
WantedBy=multi-user.target 
```

需要注意的是，代理服务器不能运行在本机上，否则会造成路由回环问题，可以运行在宿主机上，通过host-only网卡提供可靠连接。

5. 启动tun2socks服务
```bash
sudo systemctl start tun2socks
```

此时，（几乎）所有流量就都会经过代理了。

## 配置DNS
在特色网络环境下，往往存在严重的DNS污染，即便配置了上述全局代理，如果使用的是不可靠的DNS解析服务，也会存在问题，因此还需要配置DNS。

以ubuntu 24.04 server版为例，DNS被systemd-resolved接管，配置起来会比较麻烦，难以完全避免使用DHCP下发的本地DNS服务器。简单起见，可以直接禁用systemd-resolved：
```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo systemctl mask systemd-resolved
```

然后删除`/etc/resolve.conf`（实际上是一个软连接），再手动创建`/etc/resolve.conf`（真实文件），在其中配置可靠DNS服务器：
```text
nameserver 8.8.8.8
```
## 其他问题
1. vmware挂起客户机/继续运行虚拟机之后，会出现**docker容器无法访问外界网络，docker0网卡没有地址**。原因是vmware默认是软挂起，在挂起前会执行脚本释放地址，恢复后再重新通过dhcp获取地址，从而避免恢复后地址冲突([参考vmware论坛的讨论](https://community.broadcom.com/vmware-cloud-foundation/communities/community-home/digestviewer/viewthread?MessageKey=119af62b-d410-4dde-9b0d-fd84f352b95c&CommunityKey=fb707ac3-9412-4fad-b7af-018f5da56d9f#bm119af62b-d410-4dde-9b0d-fd84f352b95c))，但docker0的地址不是dhcp分配的，此时就会失去地址导致容器网络不通。解决方法：将vmware的挂起改为硬挂起（虚拟机设置-选项-电源-`挂起客户机`改为`挂起`）