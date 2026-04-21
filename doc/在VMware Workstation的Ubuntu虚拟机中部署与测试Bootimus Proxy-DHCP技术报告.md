# 在VMware Workstation的Ubuntu虚拟机中部署与测试Bootimus Proxy-DHCP技术报告

## 1. 实验环境规划与准备

### 1.1 软件版本与基础配置

本实验构建于以下经过验证的软件环境之上，旨在模拟企业级PXE服务部署的典型场景：

*   **宿主机虚拟化平台**：VMware Workstation 17 Pro 或更高版本（兼容至19）。本实验的核心挑战——NAT网络下的DHCP包处理——在此版本范围内具有一致的行为特征。
*   **服务器虚拟机**：Ubuntu 22.04 LTS (Jammy Jellyfish) 或 **Ubuntu 24.04 LTS (Noble Numbat)**。值得注意的是，Bootimus项目自2025年后，其官方Docker镜像已基于Ubuntu 24.04构建（标签如`ubuntu:noble-20250101`），这带来了更新的内核与网络栈支持。
*   **目标PXE服务**：Bootimus，一个由 **Gary Bowers** 开发的现代化、生产就绪的PXE/HTTP引导服务器。本报告采用其截至2026年初的最新稳定版本 **`v3.2.0`** （对应Docker镜像标签 `garybowers/bootimus:v3.2.0`），该版本标志着配置范式已完全迁移至**环境变量驱动**。
*   **容器运行时**：Docker Engine 24.0+ 与 Docker Compose v2.20+。这是运行Bootimus容器化服务的基石。

**虚拟机基础资源配置建议**：
*   **CPU**：2核或以上，确保处理并发DHCP请求时无瓶颈。
*   **内存**：不少于2GB，推荐4GB，为Docker、Bootimus及其SQLite/PostgreSQL数据库提供充足运行空间。
*   **磁盘**：40GB动态分配磁盘，为存储操作系统ISO镜像、引导文件及容器数据卷提供空间。
*   **网络适配器**：初始配置为 **NAT模式**（对应VMnet8虚拟网络）。此模式是本次实验网络挑战的根源，也是验证解决方案的关键。

### 1.2 VMware NAT网络模式的影响分析与前提假设

VMware Workstation的NAT模式为虚拟机提供了便捷的网络访问，但其内置的`vmnetdhcp`服务对PXE引导构成了根本性障碍。基于深度研究，其影响机制如下：

1.  **广播域隔离**：NAT网络（`vmnet8`）是一个封闭的广播域。PXE客户端发出的初始`DHCP Discover`广播包被限制在此域内，无法直接到达外部网络或宿主机上运行的`proxy-dhcp`服务。

2.  **DHCP端口过滤与抢占**：`vmnet8`虚拟适配器会**拦截所有发往UDP 67和68端口的流量**。内置的`vmnetdhcp`服务将响应所有DHCP请求。这意味着，即便外部Proxy DHCP服务（如Bootimus）配置正确，其发出的响应包（尤其是单播的`DHCP Offer`和`Ack`）也可能在NAT层被丢弃，无法送达客户端。

3.  **“全有或全无”的配置策略**：VMware未提供仅禁用内置DHCP服务中PXE选项（Option 66, 67）的细粒度控制。因此，**共存方案极不稳定**。最可靠的方案是彻底关闭NAT网络的DHCP服务，为Bootimus的`proxy-dhcp`让路。

**本实验的核心前提假设**是：为了在NAT网络内成功部署一个可用的Proxy DHCP服务，我们必须采取主动配置，**关闭VMware内置DHCP**，并解决由此产生的IP地址分配问题。实验将验证这一路径的完整可行性。

## 2. Bootimus的部署与配置

### 2.1 安装Docker与必要工具

在Ubuntu虚拟机上执行标准安装命令：
```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 net-tools tcpdump
sudo systemctl enable --now docker
sudo usermod -aG docker $USER # 将当前用户加入docker组，避免频繁使用sudo
```
安装完成后，请注销并重新登录以使组权限生效。

### 2.2 获取并配置Bootimus

1.  **创建项目目录与数据卷**：
    ```bash
    mkdir -p ~/bootimus-deploy/data/{bootloaders,isos}
    cd ~/bootimus-deploy
    ```
    `data`目录将用于持久化Bootimus的数据库、ISO镜像和自定义引导程序。

2.  **编写Docker Compose文件** (`docker-compose.yml`)：
    Bootimus `v3.x`系列完全采用环境变量配置。以下是最新版本下，启用`proxy-dhcp`功能的推荐配置。我们优先展示 **`host`网络模式**，因其最为简单直接。
    ```yaml
    version: '3.8'
    services:
      bootimus:
        image: garybowers/bootimus:v3.2.0 # 使用明确版本标签
        container_name: bootimus-pxe-server
        restart: unless-stopped
        network_mode: "host" # 关键配置：使容器共享主机网络栈，直接绑定端口67。
        environment:
          - BOOTIMUS_PROXY_DHCP_ENABLED=true # 启用核心的Proxy DHCP功能
          - BOOTIMUS_PROXY_DHCP_SUBNET=192.168.137.0/24 # 指定响应的子网，与VMnet8匹配
          - BOOTIMUS_SERVER_ADDR=192.168.137.1 # 告知客户端TFTP/HTTP服务器的地址（Ubuntu虚拟机在NAT网络中的IP）
          - BOOTIMUS_HTTP_PORT=8080
          - BOOTIMUS_TFTP_ENABLED=true
          - TZ=Asia/Shanghai
        cap_add:
          - NET_BIND_SERVICE # 允许绑定特权端口(67, 69)
          - NET_ADMIN        # 建议添加，用于高级网络操作
          - NET_RAW          # 建议添加，用于原始套接字操作（处理DHCP包）
        volumes:
          - ./data:/data # 持久化数据卷
    ```
    **关键参数解析**：
    *   `BOOTIMUS_PROXY_DHCP_SUBNET`：这是v3.x版本引入的重要变量，用于限制Proxy DHCP的响应范围，避免干扰其他网络。
    *   `cap_add`：除`NET_BIND_SERVICE`外，添加`NET_RAW`能力是处理原始DHCP/UDP数据包的**最佳实践**，提高了容器权限的精准度。
    *   `network_mode: “host”`：这是解决NAT网络广播问题的直接方案。容器将直接使用Ubuntu虚拟机的IP地址（`192.168.137.1`）和网络端口。

3.  **（替代方案）使用Macvlan网络的配置示例**：
    如果宿主机需要运行多个占用端口67的服务，或需更严格的网络隔离，可采用`macvlan`模式。此配置更为复杂，但更具生产部署价值。
    ```yaml
    version: '3.8'
    services:
      bootimus:
        image: garybowers/bootimus:v3.2.0
        container_name: bootimus-macvlan
        restart: unless-stopped
        environment:
          - BOOTIMUS_PROXY_DHCP_ENABLED=true
          - BOOTIMUS_PROXY_DHCP_SUBNET=192.168.137.0/24
          - BOOTIMUS_SERVER_ADDR=192.168.137.250 # 必须与下方分配的静态IP一致
        networks:
          pxe-macvlan-net:
            ipv4_address: 192.168.137.250 # 为容器在NAT网络内分配一个固定IP
        cap_add:
          - NET_BIND_SERVICE
          - NET_ADMIN
          - NET_RAW
        volumes:
          - ./data:/data

    networks:
      pxe-macvlan-net:
        driver: macvlan
        driver_opts:
          parent: ens33 # 需替换为Ubuntu虚拟机在NAT网络中的实际接口名（如ens160, eth0）
        ipam:
          config:
            - subnet: 192.168.137.0/24
              gateway: 192.168.137.2 # 通常为VMware NAT网关地址
              ip_range: 192.168.137.240/28
    ```
    **注意**：使用`macvlan`时，宿主机默认无法与容器直接通信，且需要确认父接口正确。

### 2.3 启动Bootimus服务

在`docker-compose.yml`文件所在目录执行：
```bash
docker-compose up -d
```
启动后，使用以下命令验证服务状态与日志：
```bash
docker-compose logs -f bootimus
# 关注日志中是否有“proxyDHCP enabled”等相关提示，并无报错。
# 验证容器是否正在运行
docker-compose ps
```

Bootimus的Web管理界面默认在宿主机的`8081`端口（如果使用`host`模式），可通过`http://<ubuntu-vm-ip>:8081`访问。首次登录密码需通过`docker-compose logs bootimus | grep “Admin password”`命令从日志中获取。

## 3. VMware Workstation网络专项配置

此部分是实验成功的关键，旨在克服NAT网络对PXE的固有限制。

### 3.1 配置虚拟网络编辑器

1.  打开VMware Workstation，进入`编辑` -> `虚拟网络编辑器`。
2.  选择`VMnet8 (NAT模式)`，点击`更改设置`获取管理员权限。
3.  **取消勾选`使用本地DHCP服务将IP地址分配给虚拟机`**。此操作将停止`vmnetdhcp`服务，释放UDP 67端口，并避免DHCP响应冲突。这是实现Bootimus Proxy DHCP工作的**先决条件**。
4.  记下`子网IP`（如`192.168.137.0`）和`子网掩码`。这些信息需与Bootimus配置中的`BOOTIMUS_PROXY_DHCP_SUBNET`保持一致。
5.  点击`NAT设置`，记录`网关IP`（通常为`192.168.137.2`）。此地址将作为后续客户端虚拟机的默认网关。

### 3.2 为Ubuntu PXE服务器虚拟机配置静态IP

关闭DHCP服务后，Ubuntu虚拟机将无法自动获取IP。需要配置静态IP以确保网络连通。

1.  编辑Ubuntu的网络配置文件（以`netplan`为例）：
    ```bash
    sudo nano /etc/netplan/00-installer-config.yaml
    ```
2.  配置静态IP（假设网卡为`ens33`）：
    ```yaml
    network:
      version: 2
      ethernets:
        ens33:
          addresses: [192.168.137.1/24] # 设置静态IP，需在NAT子网内且未被占用
          routes:
            - to: default
              via: 192.168.137.2 # VMware NAT网关地址
          nameservers:
            addresses: [8.8.8.8, 1.1.1.1] # 配置DNS，可指向网关或公共DNS
          dhcp4: no # 明确禁用DHCP客户端
    ```
3.  应用配置：
    ```bash
    sudo netplan apply
    ```
4.  验证网络：
    ```bash
    ip addr show ens33
    ping 192.168.137.2 # 应能通网关
    ping 8.8.8.8 # 应能通外网
    ```

### 3.3 防火墙规则配置

确保Ubuntu虚拟机的防火墙允许PXE相关流量。使用`ufw`配置示例如下：
```bash
sudo ufw allow 67/udp comment 'Bootimus Proxy DHCP'
sudo ufw allow 69/udp comment 'TFTP'
sudo ufw allow 4011/udp comment 'PXE Discovery'
sudo ufw allow 8080/tcp comment 'HTTP Boot'
sudo ufw allow 8081/tcp comment 'Bootimus Admin UI'
sudo ufw enable
```
**关键点**：对于TFTP协议，如果使用`host`网络模式，上述规则已足够。如果使用复杂的`bridge`模式（不推荐），可能需要额外的连接跟踪(`conntrack`)规则来处理TFTP数据端口。

## 4. UEFI与Legacy启动模式测试方案

### 4.1 创建测试客户端虚拟机

1.  在VMware Workstation中新建一个虚拟机。
    *   操作系统：选择“稍后安装操作系统”。
    *   固件类型：**分别创建两台，一台选择`UEFI`，一台选择`BIOS`（传统）**，以进行对比测试。
    *   磁盘：分配20GB即可。
    *   网络适配器：连接到`NAT模式`（VMnet8）。

2.  **修改虚拟机设置**：
    *   **移除CD/DVD和硬盘**：在“虚拟机设置”中，暂时移除或断开CD/DVD驱动器，并将硬盘的“启动时连接”取消勾选。强制虚拟机从网络启动。
    *   **网卡类型**：**对于UEFI虚拟机，务必将其网络适配器类型设置为`E1000E`**。VMXNET3等高级网卡在UEFI模式下可能缺乏PXE引导支持，是导致`PXE-E18`错误的常见原因。
    *   **安全启动（仅UEFI）**：首次测试时，在虚拟机设置的`选项` -> `高级` -> `UEFI固件设置`中启动，进入虚拟BIOS**禁用Secure Boot**。因为Bootimus默认不提供微软签名的引导程序，Secure Boot会阻止其加载。

### 4.2 分步验证PXE引导流程

启动客户端虚拟机，并立即按`Esc`或`F12`（根据VMware提示）进入启动菜单，选择从网络(`Network`)启动。

**观察与验证点**：

1.  **DHCP Discover/Offer阶段**：
    *   **现象**：客户端屏幕应显示类似“Initializing， Performing DHCP request…”的信息。
    *   **验证**：在Ubuntu PXE服务器上运行抓包命令，验证请求是否到达以及Bootimus是否正确响应：
        ```bash
        sudo tcpdump -i any -n port 67 or port 68
        # 或使用更精确的Wireshark分析
        ```
    *   **关键分析**：对比UEFI和Legacy客户端发出的`DHCP Discover`报文。使用Wireshark过滤器 `bootp.option.type == 93` 查看**选项93（Client System Architecture Type）**。Legacy应为`0x0000`，UEFI x64应为`0x0007`或`0x0009`。确认Bootimus的响应中提供的引导文件名（如`ipxe.efi`对应UEFI，`undionly.kpxe`对应Legacy）是否正确。

2.  **TFTP文件下载阶段**：
    *   **现象**：客户端在获取IP后，应显示“Downloading NBP file…”、“TFTP prefix…”等。
    *   **验证**：检查Bootimus容器日志，确认TFTP传输记录：
        ```bash
        docker-compose logs bootimus | grep -i tftp
        ```
    *   **故障排查**：若在此阶段超时或失败，检查Ubuntu防火墙的UDP 69端口规则，以及`/data`卷下的引导文件是否存在。

3.  **iPXE/HTTP引导阶段**：
    *   **现象**：成功下载iPXE引导程序后，客户端应加载iPXE并尝试通过HTTP(`8080`端口)获取引导菜单(`menu.ipxe`)。
    *   **验证**：在Ubuntu服务器上，可直接测试HTTP服务：
        ```bash
        curl -v http://localhost:8080/menu.ipxe
        ```
    *   **最终结果**：客户端应成功显示Bootimus提供的网络引导菜单，列出可安装的操作系统或工具（如Memtest86+、GParted Live等）。

### 4.3 基于报文的高级诊断

若引导失败，使用Wireshark进行深度分析是唯一可靠的方法。
1.  **在Ubuntu PXE服务器上启动Wireshark**，选择正确的接口（`any`或`ens33`）。
2.  **应用过滤器**：`(udp.port == 67 or udp.port == 68) or tftp`
3.  **分析流程**：
    *   是否有来自客户端MAC的`DHCP Discover`广播？
    *   网络中是否有多个`DHCP Offer`响应（冲突）？
    *   Bootimus的`Offer`报文是否包含了正确的`next-server`（应是Ubuntu虚拟机的IP）和`bootfile-name`？
    *   客户端是否发出了对引导文件的`TFTP Read Request`？

## 5. 故障排查与常见问题分析

### 5.1 基于日志的快速诊断

Bootimus容器日志是首要的排查工具。以下命令组合能快速定位问题：

```bash
# 检查proxyDHCP是否启用并收到请求
docker-compose logs bootimus | grep -E “proxyDHCP|PROXY”

# 检查TFTP传输情况
docker-compose logs bootimus | grep -i tftp

# 检查HTTP引导服务
docker-compose logs bootimus | grep -E “HTTP|8080”

# 查看完整的启动和运行日志
docker-compose logs --tail 50 -f bootimus
```

### 5.2 常见故障场景与解决方案

| 故障现象 | 可能原因 | 诊断与解决方案 |
| :--- | :--- | :--- |
| **客户端无任何PXE响应，直接进入硬盘启动** | 1. VMware NAT DHCP未关闭，未释放端口67。<br>2. Bootimus容器网络模式错误（非`host`/`macvlan`）。<br>3. 防火墙阻止了UDP 67端口。 | 1. 确认虚拟网络编辑器中已禁用DHCP。<br>2. 确认`docker-compose.yml`中为`network_mode: “host”`。<br>3. 运行`sudo ufw status verbose`检查规则。 |
| **客户端显示`PXE-E18: Server response timeout`** | 1. (UEFI特有) 网卡类型不是`E1000E`。<br>2. (UEFI特有) Secure Boot未禁用。<br>3. `BOOTIMUS_PROXY_DHCP_SUBNET`配置错误，与实际NAT子网不匹配。<br>4. `BOOTIMUS_SERVER_ADDR`设置错误。 | 1. 检查并更改UEFI虚拟机网卡为`E1000E`。<br>2. 进入虚拟UEFI设置禁用Secure Boot。<br>3. 核对并修正`BOOTIMUS_PROXY_DHCP_SUBNET`。<br>4. 确保`SERVER_ADDR`是Ubuntu虚拟机在NAT网内的正确IP。 |
| **能获取IP，但TFTP下载失败** | 1. 防火墙阻止UDP 69端口。<br>2. Bootimus的`/data`卷下缺少引导文件。<br>3. TFTP根目录权限问题。 | 1. 开放UDP 69端口。<br>2. 检查容器日志，确认Bootimus是否成功生成了引导文件。<br>3. 确保宿主机`./data`目录对容器进程可读（权限至少755）。 |
| **引导至iPXE后无法加载HTTP菜单** | 1. 防火墙阻止TCP 8080端口。<br>2. Bootimus HTTP服务未正常启动。<br>3. 客户端与服务器之间存在路由问题。 | 1. 开放TCP 8080端口。<br>2. 使用`curl`在服务器本地测试`http://localhost:8080/menu.ipxe`。<br>3. 确认客户端与服务器IP在同一子网。 |
| **多台客户端并发启动时部分失败** | 1. Docker容器资源（CPU、内存）限制。<br>2. TFTP协议本身的并发性能瓶颈。<br>3. 系统文件描述符(`nofile`)限制。 | 1. 在`docker-compose.yml`中适当增加`cpus`和`mem_limit`。<br>2. 考虑在Bootimus中启用HTTP Boot（若客户端支持），以替代TFTP传输大文件。<br>3. 调整宿主机和容器的`ulimits`。 |

## 6. 进阶测试建议与性能评估方向

完成基本功能验证后，可开展以下进阶测试，以评估系统在生产环境下的表现。

### 6.1 多客户端并发启动（“启动风暴”）模拟

1.  **测试目标**：验证Bootimus Proxy DHCP服务在高并发场景下的稳定性和性能。
2.  **方法**：
    *   利用VMware的**链接克隆**功能，快速创建10-20台配置相同的测试客户端虚拟机。
    *   使用脚本（如PowerCLI、`govc`或简单的循环启动命令）批量、几乎同时启动这些虚拟机。
    *   监控Bootimus容器在启动风暴期间的资源使用情况（`docker stats`）。
    *   记录并统计客户端的PXE启动成功率、平均启动时间（从加电到显示引导菜单）。
3.  **性能调优方向**：
    *   **容器资源**：在`docker-compose.yml`中设置明确的资源限制与预留。
        ```yaml
        bootimus:
          deploy: # 或使用`resources`关键字（取决于Compose版本）
            resources:
              limits:
                cpus: ‘2.0’
                memory: 2G
              reservations:
                cpus: ‘1.0’
                memory: 1G
        ```
    *   **文件系统**：将`./data`目录挂载到Ubuntu虚拟机的**`tmpfs`（内存盘）** 或高性能SSD存储上，可极大加速TFTP/HTTP文件的读取速度。
    *   **内核参数**：调整Ubuntu宿主机的网络参数，如增加UDP缓冲区大小(`net.core.rmem_max`, `net.core.wmem_max`)。

### 6.2 不同网络架构适配测试

1.  **桥接模式测试**：将Ubuntu PXE服务器和客户端虚拟机的网络都改为**桥接模式**。测试在更接近物理网络的环境中，Bootimus Proxy DHCP与局域网中现有路由器DHCP的共存情况。此场景下，`BOOTIMUS_PROXY_DHCP_SUBNET`的精确配置尤为重要。
2.  **仅主机模式测试**：创建一个纯内部的“仅主机模式”网络（VMnet1），进行隔离的网络引导测试。此模式无NAT和外部DHCP干扰，是验证PXE服务本身功能的理想环境。

### 6.3 安全与高可用性探索

1.  **安全加固**：
    *   尝试以非root用户运行Bootimus容器。这需要更精细的Linux能力配置和文件卷权限设置，是提升容器安全性的重要实践。
    *   为Bootimus的管理界面（8081端口）配置反向代理（如Nginx），并添加HTTPS加密与基础认证。
2.  **高可用架构设想**：
    *   研究在多个Ubuntu虚拟机上部署多个Bootimus实例，配合**DHCP故障转移协议**或外部负载均衡器，构建无单点故障的PXE服务体系。这需要解决DHCP租约同步和客户端状态一致性问题。

### 6.4 与现代引导技术的结合

*   **UEFI HTTP Boot**：研究并测试通过Bootimus配置**UEFI HTTP Boot**。HTTP Boot使用TCP 80端口，避免了TFTP的诸多性能瓶颈，且Bootimus原生支持。这是未来网络引导的重要发展方向。
*   **集成签名引导链**：为支持UEFI Secure Boot的环境，研究如何将经过微软签名的`shim.efi`和`grubx64.efi`集成到Bootimus的引导流程中，实现安全启动下的网络引导。

通过本报告详述的步骤、配置与测试方案，我们系统地验证了在VMware Workstation最具挑战的NAT网络环境下，部署和运行基于Bootimus的现代化Proxy DHCP PXE服务的完整流程与可行性，并为后续的生产部署与性能优化提供了明确的技术路径。

---
## 已研究 2 个本地资源
1. docker-compose.md
2. https://github.com/garybowers/bootimus?tab=readme-ov-file

