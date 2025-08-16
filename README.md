# Clash-for-Linux
 Implementing proxy configuration in Linux system

### 一、引言
作者最近有一个项目的代码中需要频繁访问外网（该代码在服务器上运行），本来是通过笔记本的clash for windows开局域网代理让服务器访问我笔记本的端口实现的，但是这种方式有以下两个**弊端**：

> 1. 笔记本不能关机，否则服务器无法访问到我笔记本的端口
> 2. 我的笔记本必须要使用服务器所在局域网


以上两个弊端束缚住了作者的笔记本，遂打算研究一下如何将clash for linux配置进服务器里面，让代码去访问服务器映射出来的端口即可

注：以下解决路径是需要自备代理（节点），只是讲述如何配置

### 二、问题解决路径
##### 下载clash for linux
clash for linux其实和clash for windows一样，是一个帮你处理代理（节点）的一个服务，当然一些高手可以直接用一些工具来自己搭建服务，作者这里还是选择了大家都能使用上的clash服务
```bash
https://github.com/Kuingsmile/clash-core/releases
```
[clash下载地址](https://github.com/Kuingsmile/clash-core/releases)
(大家可以点击上面的链接去寻找适合自己的clash版本，一般64位linux系统建议选择)

[clash-linux-amd64-v1.18.0.gz](https://github.com/Kuingsmile/clash-core/releases/download/1.18/clash-linux-amd64-v1.18.0.gz)

下载好后将其移动到一个clash文件夹下准备进行解压
```bash
gzip -d clash-linux-amd64-v1.18.0.gz   #对下载内容进行解压
```
接下来可以解压后的文件改名为clash_meta(自由选择，但后续需要统一）
```bash
sudo mv clash_meta /usr/local/bin/    #将clash_meta移入usr文件夹实现全局管理  
sudo chmod +x /usr/local/bin/clash_meta     #给clash_meta配置权限
```
### 三、给clash for windows配置节点
在一些比较正规的节点网站可以下载clash配置文件，大致命令如下
```bash
mkdir -p ~/.config/clash #创建一个存放config.yml的文件夹
wget -O config.yml "http://xxxxxx/link/xxxxxxxf?clash=1&log-level=info" 
#用于拉取节点网站配置好的config

mv config.yml ~/.config/clash/config.yaml #将config文件移动到专门存放config的文件夹中
```

此时可以通过clash_meta命令尝试启动代理
```bash
clash_meta  #和前面设置的文件名字一致
```
如果出现以下问题
```bash
INFO[0000] Can't find MMDB, start download 
FATA[0000] Initial configuration directory error: can't initial MMDB: can't download MMDB: Get
```
本质是Clash.Meta 启动时会尝试下载 GeoIP 数据库（Country.mmdb） 来判断流量国家，但你现在网络环境没有直连网络，所以下载失败了（connection refused）

可以尝试用一些国内镜像源，或者直接从官方拉取
```bash
#建议开一个文件夹存放以下内容
mkdir -p ~/.config/clash/GeoIP

wget -O ~/.config/clash/GeoIP/Country.mmdb https://github.com/Dreamacro/maxmind-geoip/releases/latest/download/Country.mmdb
wget -O ~/.config/clash/GeoIP/GeoSite.dat https://github.com/Dreamacro/clash-geodata/releases/latest/download/GeoSite.dat
```
我们需要将自己下载的geoip内容写进配置文件中，nano用于打开该文件
```bash
nano ~/.config/clash/config/config.yaml  #如果没有权限请用sudo
```
往配置文件（config.yaml）中加入内容
```bash
geoip: ~/.config/clash/GeoIP/Country.mmdb
geosite: ~/.config/clash/GeoIP/GeoSite.dat
```
最终启动服务就好啦！
```bash
clash_meta -d ~/.config/clash  #服务启动命令
```
可以测试是否成功配置，在保持上一个命令行窗口不关闭的同时打开一个新的终端
```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890  
#以上用于配置代理，让终端走本地映射到7890的端口（clash默认）
```
抓取Google页面尝试
```bash
curl -x http://127.0.0.1:7890 https://www.google.com #http的代理模式
```
如果成功返回一大串字符内容即说明代理成功配置进Linux系统啦！

### 四、如何让将clash配置成系统服务
如果尝试完上述内容，你会发现如果你Ctrl+C掉启动命令，代理无法继续生效，这其实是clash的生命周期的问题，他需要保持启动你才能使用代理，但是一直开一个命令行或者每次启动都输入启动命令并不是我们想要的，有什么办法能让他一直启动着呢？

方案一：nohup挂在后台

方案二：配置成系统服务（推荐）

以下是将clash for linux配置进系统服务的方式（要求7890，7891，7892端口不被占用）

创建systemd服务文件
```bash
sudo nano /etc/systemd/system/clash.service
```
 写入以下内容
```bash
[Unit]
Description=Clash Meta Service
After=network.target

[Service]
Type=simple
#User和Group填写当前用户
User=xxx
Group=xxx
ExecStart=/usr/local/bin/clash_meta -d /home/ices/.config/clash  
#ExectStart填写之前的两个文件路径
Restart=on-failure
RestartSec=5s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```
开始启动系统服务
```bash
# 保存后刷新 systemd 配置
sudo systemctl daemon-reload

# 启动服务
sudo systemctl start clash

# 查看状态
sudo systemctl status clash
```
如果希望能够开机自启动
```bash
# 设置开机自启
sudo systemctl enable clash
```
可以按照第二大点的测试方法尝试服务是否成功配置好了代理

​
