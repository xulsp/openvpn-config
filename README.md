
## 服务端配置
```bash
# lzop 用于数据压缩数据，非必要
apt-get install openvpn easy-rsa lzop

cd /usr/share/easy-rsa
cp openssl-1.0.0.cnf openssl.cnf
source ./vars
# 清除一次keys文件夹下的证书
./clean-all
# 生成根证书 ca.crt ca.key
./build-ca
./build-key-server --batch server
# 生成dh2048.pem
./build-dh
# 生成ta.key
openvpn --genkey --secret keys/ta.key

mkdir -p /etc/openvpn/keys
cd keys
cp ca.crt /etc/openvpn/keys/
cp server.crt /etc/openvpn/keys/
cp server.key /etc/openvpn/keys/
cp dh2048.pem /etc/openvpn/keys/
cp ta.key /etc/openvpn/keys/

# 拷贝服务器配置
cd /usr/share/doc/openvpn/examples/sample-config-files/
cp server.conf.gz /etc/openvpn/
cd /etc/openvpn/
mkdir ccd
gzip -d server.conf.gz
```
### 编辑server.conf
vim server.conf
```
# openvpn监听的端口
port 1194
# 使用的传输协议
proto tcp
dev tun
# 这里仅为主要配置，做参考
# 证书路径
ca keys/ca.crt
cert keys/server.crt
key keys/server.key

dh keys/dh2048.pem

# ccd是一个目录用于指定静态ip,不需要的话可以不要这一行
client-config-dir ccd
# vpn的虚拟网段，可以自定义
server 10.8.0.0 255.255.255.0
# 会指定客户端有哪些ip
ifconfig-pool-persist /etc/openvpn/ipp.txt

# 这里的ip和mask 取服务器内网网卡的ip和mask
push "route 172.16.3.11 255.255.255.0"

# 服务端为0，客户端为1
tls-auth keys/ta.key 0

# 允许最大的客户端数
max-clients 100

persist-key
persist-tun

# 日志输出
status /var/log/openvpn/openvpn-status.log
log         /var/log/openvpn/openvpn.log
log-append  /var/log/openvpn/openvpn.log

```
### 启动openvpn
```bash
#设置开机启动
systemctl enable openvpn@server
#启动openvpn
systemctl start openvpn@server
# 检查是否正常启动
ps -ef | grep 'openvpn'
# 查看网卡是否多了一个ip以 10.8开头的网卡
```

## 配置客户端
```bash
vi /etc/openvpn/ipp.txt
# ipp.txt 格式如下,名称和ip地址
old,10.8.0.6
new,10.8.0.211

cd /etc/openvpn/ccd
# 文件名为 一个特殊名称,生成证书会用到
vi old
#old文件中填入内容
ifconfig-push 10.8.0.6 255.255.255.0


#生成客户端证书
cd /usr/share/easy-rsa/
source ./vars
./build-key old #一路回车

mkdir -p ~/openvpn-client
# 拷贝证书
cp ./keys/old.* ~/openvpn-client/

# 拷贝配置
cd /usr/share/doc/openvpn/examples/sample-config-files/
cp client.conf ~/openvpn-client/client.ovpn
# 编辑配置，内容如下
cd ~/openvpn-client/
```
### vi client.ovpn
```
# client.opvn配置内网
# 这里表示是客户端
client


dev tun
proto tcp

# 服务端的公网ip 和端口
remote 14.153.76.90 1194

resolv-retry infinite

nobind

persist-key
persist-tun

# 这里设置证书和秘钥
ca ca.crt
cert client.crt
key client.key

remote-cert-tls server

# 服务端为0，客户端为1
tls-auth ta.key 1

# 是否启用lzop压缩
comp-lzo
verb 3

# 输出日志 方便排错
log         /var/log/openvpn/openvpn.log
log-append  /var/log/openvpn/openvpn.log
```

拷贝openvpn-client目录到客户端主机上
```bash
apt-get install openvpn lzop
#可以拷贝到/opt/目录下
# 启动客户端
openvpn --cd /opt/openvpn-client --daemon --config client.ovpn

# 验证 , ping下服务端地址
ping 10.8.0.1
# ifconfig 查看是否生成以10.8开头的网卡
```
