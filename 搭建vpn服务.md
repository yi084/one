

------

### **1. 准备工作**

1. 确保系统已更新：

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. 确认你有`sudo`权限（或以`root`用户操作）。

3. 记录你的公网IP地址（例如通过`ipinfo.io`获取）。

------

### **2. 安装WireGuard**

1. 安装WireGuard软件包：

   ```bash
   sudo apt install wireguard -y
   ```

2. 安装完成后，确认是否安装成功：

   ```bash
   sudo systemctl status wg-quick@wg0
   ```

------

### **3. 配置WireGuard**

#### **(1) 生成密钥对**

1. 创建密钥目录并生成服务器密钥：

   ```bash
   mkdir -p /etc/wireguard
   cd /etc/wireguard
   umask 077
   wg genkey | tee server_private.key | wg pubkey > server_public.key
   ```

2. 生成客户端密钥对：

   ```bash
   wg genkey | tee client_private.key | wg pubkey > client_public.key
   ```

#### **(2) 配置服务器端WireGuard**

1. 创建配置文件`/etc/wireguard/wg0.conf`：

   ```bash
   sudo nano /etc/wireguard/wg0.conf
   ```

   填入以下内容：

   ```ini
   [Interface]
   Address = 10.0.0.1/24
   SaveConfig = true
   ListenPort = 51820
   PrivateKey = <服务器私钥内容> # 使用`cat server_private.key`获取内容
   
   # 开启NAT和流量转发
   PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE+
   
   PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
   ```

   替换`<服务器私钥内容>`为你的服务器私钥。

2. 添加客户端配置：

   ```ini
   [Peer]
   PublicKey = <客户端公钥内容> # 使用`cat client_public.key`获取内容
   AllowedIPs = 10.0.0.2/32
   ```

#### **(3) 启用IP转发**

1. 编辑

   ```
   /etc/sysctl.conf
   ```

   ：

   ```bash
   sudo nano /etc/sysctl.conf
   ```

   取消以下行的注释：

   ```
   net.ipv4.ip_forward=1
   ```

2. 使配置生效：

   ```bash
   sudo sysctl -p
   ```

#### **(4) 启动WireGuard服务**

1. 启动服务：

   ```bash
   sudo systemctl start wg-quick@wg0
   ```

2. 设置开机自启：

   ```bash
   sudo systemctl enable wg-quick@wg0
   ```

------

### **4. 配置客户端WireGuard**

1. 在客户端设备上安装WireGuard（例如Windows、macOS、Linux、Android或iOS）。

2. 创建客户端配置文件，例如

   ```
   client.conf
   ```

   ：

   ```ini
   [Interface]
   Address = 10.0.0.2/24
   PrivateKey = <客户端私钥内容> # 使用`cat client_private.key`获取内容
   
   [Peer]
   PublicKey = <服务器公钥内容> # 使用`cat server_public.key`获取内容
   Endpoint = <服务器公网IP>:51820
   AllowedIPs = 0.0.0.0/0
   ```

3. 将客户端配置文件导入客户端WireGuard应用。

------

### **5. 验证VPN是否成功搭建**

#### **(1) 测试客户端连接**

- 在客户端设备上启用VPN连接。
- 确认连接成功后，检查是否可以访问服务器的VPN地址`10.0.0.1`。

#### **(2) 验证外网流量是否通过VPN**

1. 使用客户端访问 [WhatIsMyIP](https://whatismyipaddress.com/)。
2. 确认显示的IP为服务器的公网IP。

#### **(3) 测试内部Ping**

- 从客户端执行：

  ```bash
  ping 10.0.0.1
  ```

- 从服务器执行：

  ```bash
  ping 10.0.0.2
  ```

------

### **常见问题**

1. 端口未开放

   ：确认服务器的防火墙允许

   ```
   51820/udp
   ```

   端口。

   ```bash
   sudo ufw allow 51820/udp
   ```

2. **客户端无法连接**：检查配置文件中`PublicKey`和`PrivateKey`是否正确匹配。

3. **流量未转发**：确认`iptables`规则是否正确。
