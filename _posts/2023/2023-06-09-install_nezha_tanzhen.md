---
title: "手把手教你安装哪吒探针"
date: 2023-06-08 
author: other
category: [2023.]
tags: [vps]
permalink: /:year/:month-:day/01
---

[教程]手把手教你安装哪吒探针

# 目录
 1. 前言
 2. 土味概念解释
 3. 前期准备
 3. Github OAuth设置
 5. 面板安装
 6. 客户端安装
 7. Alpine系统手动安装客户端
 8. 配置Nginx反代
 9. 申请SSL证书
 10. 反代后修改Github Oauth设置
 11. 常见坑点
# 1. 前言
 玩鸡半年多，手里的小鸡渐渐多了起来，有了监控的需求，最后选择了哪吒探针。
 这个探针很好用，但是文档与群组的支持稍显不足，遇到了一些坑，但最后都成功解决了，于是有了本教程，希望大家在搭建自己探针的时候，少走弯路。
# 2. 土味概念解释
 |概念|解释|
 |----|---|
 |面板机|1台小鸡，作为主控端，用于展示探针界面。|
 |客户机|N台小鸡，作为被控端，安装有哪吒探针的agent，与面板机通信，上传小鸡信息。|
 |面板端口|访问探针界面使用的端口，默认8008|
 |RPC端口|agent与面板机通信用的端口，默认5555|
 |面板域名|访问探针界面使用的域名，解析到面板机ip，开启CDN（小云朵），保护ip不泄漏|
 |通信域名|agent与面板机通信用的域名，解析到面板机ip，不开CDN|
 |Github OAuth|登录面板后台使用Github的身份，称为"第三方登录"|
 
# 3. 前期准备
 |名称|数量|说明|
 |---|----|---|
 |面板机|1||
 |客户机|N||
 |面板域名|1|做好A记录解析，开启CDN，下文以a.com代替|
 |通信域名|1|做好A记录解析，下文以b.com代替|
 |Github账户|1||
 |常用命令||curl、wget、sudo|
 
 # 4. Github OAuth设置
 1. 访问`https://github.com/settings/developers`，右上角`New OAuth App`
 2. 如图设置应用名称、主页、回调页面
 这里借用一下通信域名b.com，后面会改成面板域名
 ![Snipaste_2023-06-07_12-13-05](https://i2.100024.xyz/2023/06/07/k2e7wx.webp)
 
 # 5. 面板安装
 1. 官方安装脚本一把梭
 `curl -L https://raw.githubusercontent.com/naiba/nezha/master/script/install.sh  -o nezha.sh && chmod +x nezha.sh && sudo ./nezha.sh`
 2. 境内服务器使用：
 `curl -L https://cdn.jsdelivr.net/gh/naiba/nezha@master/script/install.sh -o nezha.sh && chmod +x nezha.sh && sudo CN=true ./nezha.sh`
 
 # 6. 客户端安装
 1. 访问面板
 `http://b.com:8008`
 2. 点击右上角登录，使用Github账户登录，进入后台
 ![Snipaste_2023-06-07_12-24-09](https://i2.100024.xyz/2023/06/07/k9ennt.webp)
 3. 以Linux系统为例，先点击添加服务器，设置好信息，然后点击绿色小企鹅，复制一键安装客户端命令，在客户机上粘贴执行即可
 命令格式为：
 `curl -L https://raw.githubusercontent.com/naiba/nezha/master/script/install.sh -o nezha.sh && chmod +x nezha.sh && sudo ./nezha.sh install_agent b.com 5555 [密钥]`
 4. 一切顺利的话，在面板首页即可看到客户端上线了
 ![Snipaste_2023-06-07_12-29-37](https://i2.100024.xyz/2023/06/07/kcegvu.webp)
 
 # 7. Alpine系统手动安装客户端
 我尝试过官方一键脚本，但是不好使，这里我们手动安装，以amd64架构的小鸡为例
 1. 下载agent：
 访问`https://github.com/naiba/nezha/releases`，根据小鸡架构选择软件包
 下载`wget https://github.com/naiba/nezha/releases/download/v0.14.12/nezha-agent_linux_amd64.zip`
 
 2. 下面配置服务，需要按实际情况修改：
 ```
 SERVER="b.com:5555" #通信域名:RPC端口
 SECRET="xxxxxxxxxx" #密钥，在面板后台点击复制
 ```
 3. 配置服务：
 ```
 vim /etc/init.d/nezha-agent
 chmod +x /etc/init.d/nezha-agent
 ```
 ```
 #!/sbin/openrc-run
 SERVER="b.com:5555" #通信域名:RPC端口
 SECRET="xxxxxxxxxx" #密钥，在面板后台点击复制
 TLS="" # 是否启用 tls 是 "--tls" 否留空
 NZ_BASE_PATH="/opt/nezha" #agent下载目录，可以自由配置
 NZ_AGENT_PATH="${NZ_BASE_PATH}/agent"
 pidfile="/run/${RC_SVCNAME}.pid"
 command="${NZ_AGENT_PATH}/nezha-agent"
 command_args="-s ${SERVER} -p ${SECRET} ${TLS}"
 command_background=true
 depend() {
 	need net
 }
 
 checkconfig() {
 	GITHUB_URL="github.com"
 	if [ ! -f "${NZ_AGENT_PATH}/nezha-agent" ]; then
 		if [[ $(uname -m | grep 'x86_64') != "" ]]; then
 			os_arch="amd64"
 		elif [[ $(uname -m | grep 'i386\|i686') != "" ]]; then
 			os_arch="386"
 		elif [[ $(uname -m | grep 'aarch64\|armv8b\|armv8l') != "" ]]; then
 			os_arch="arm64"
 		elif [[ $(uname -m | grep 'arm') != "" ]]; then
 			os_arch="arm"
 		elif [[ $(uname -m | grep 's390x') != "" ]]; then
 			os_arch="s390x"
 		elif [[ $(uname -m | grep 'riscv64') != "" ]]; then
 			os_arch="riscv64"
 		fi
 		local version=$(curl -m 10 -sL "https://api.github.com/repos/naiba/nezha/releases/latest" | grep "tag_name" | head -n 1 | awk -F ":" '{print $2}' | sed 's/\"//g;s/,//g;s/ //g')
 		if [ ! -n "$version" ]; then
 			version=$(curl -m 10 -sL "https://fastly.jsdelivr.net/gh/naiba/nezha/" | grep "option\.value" | awk -F "'" '{print $2}' | sed 's/naiba\/nezha@/v/g')
 		fi
 		if [ ! -n "$version" ]; then
 			version=$(curl -m 10 -sL "https://gcore.jsdelivr.net/gh/naiba/nezha/" | grep "option\.value" | awk -F "'" '{print $2}' | sed 's/naiba\/nezha@/v/g')
 		fi
 		if [ ! -n "$version" ]; then
 			echo -e "获取版本号失败，请检查本机能否链接 https://api.github.com/repos/naiba/nezha/releases/latest"
 			return 0
 		else
 			echo -e "当前最新版本为: ${version}"
 		fi
 		wget -t 2 -T 10 -O nezha-agent_linux_${os_arch}.zip https://${GITHUB_URL}/naiba/nezha/releases/download/${version}/nezha-agent_linux_${os_arch}.zip >/dev/null 2>&1
 		if [[ $? != 0 ]]; then
 			echo -e "Release 下载失败，请检查本机能否连接 ${GITHUB_URL}${plain}"
 			return 0
 		fi
 		mkdir -p $NZ_AGENT_PATH
 		chmod 755 -R $NZ_AGENT_PATH
 		unzip -qo nezha-agent_linux_${os_arch}.zip && mv nezha-agent $NZ_AGENT_PATH && rm -rf nezha-agent_linux_${os_arch}.zip README.md
 	fi
 	if [ ! -x "${NZ_AGENT_PATH}/nezha-agent" ]; then
 		chmod +x ${NZ_AGENT_PATH}/nezha-agent
 	fi
 }
 start_pre() {
 	if [ "${RC_CMD}" != "restart" ]; then
 		checkconfig || return $?
 	fi
 }
 ```
 4. 启动agent并设置开机自启
 ```
 rc-service nezha-agent start
 rc-update add nezha-agent
 ```
 
 # 8. 配置Nginx反代
 将面板机的8008端口反代到`https://a.com`
 1. 需要按实际情况修改下列信息
 ```
 server_name a.com; # 面板域名
 
 ssl_certificate          /etc/nezha/server.crt; # 你的域名证书路径
 ssl_certificate_key      /etc/nezha/server.key; # 你的域名私钥路径
 ```
 2. nginx配置：
 ```
 user www-data;
 worker_processes auto;
 pid /run/nginx.pid;
 include /etc/nginx/modules-enabled/*.conf;
 
 events {
     worker_connections 1024;
 }
 
 http {
     sendfile on;
     tcp_nopush on;
     tcp_nodelay on;
     keepalive_timeout 65;
     types_hash_max_size 2048;
 
     include /etc/nginx/mime.types;
     default_type application/octet-stream;
     gzip on;
 
     access_log /root/nginx.log;
 
     server {
         listen 443 ssl http2;
         listen [::]:443 ssl http2;
         server_name a.com; 
 
         ssl_certificate          /etc/nezha/server.crt; # 你的域名证书路径
         ssl_certificate_key      /etc/nezha/server.key; # 你的域名私钥路径
 
         underscores_in_headers on;
 
         location / {
             proxy_pass http://127.0.0.1:8008;
             proxy_set_header Host $http_host;
             proxy_set_header      Upgrade $http_upgrade;
         }
         location ~ ^/(ws|terminal/.+)$  {
             proxy_pass http://127.0.0.1:8008;
             proxy_http_version 1.1;
             proxy_set_header Upgrade $http_upgrade;
             proxy_set_header Connection "Upgrade";
             proxy_set_header Host $http_host;
         }
     }
 
     server {
         listen 80;
         location /.well-known/ {
                root /var/www/html;
             }
         location / {
                 rewrite ^(.*)$ https://$host$1 permanent;
             }
     }
 }
 ```
 
 # 9. 申请SSL证书
 这里使用acme.sh申请a.com的SSL证书，Webroot模式
 >原理是acme.sh会在网站对应域名的webroot目录下生成域名验证文件, 然后通过公网访问之，以验证对域名的所有权
 
 1. 面板机上安装acme.sh
 `curl https://get.acme.sh | sh -s email=my@example.com`
 2. 切换CA机构
 `acme.sh --set-default-ca --server letsencrypt`
 3. 申请证书，按实际情况修改面板域名
 `acme.sh --issue -d a.com -k ec-256 --webroot /var/www/html`
 4. 安装证书，按实际情况修改面板域名和证书路径
 `acme.sh --install-cert -d a.com --ecc --key-file /etc/nezha/server.key --fullchain-file /etc/nezha/server.crt --reloadcmd "systemctl force-reload nginx"`
 
 # 10. 反代后修改Github Oauth设置
 需要将http修改为https，主页和回调页面修改成面板域名，不能是原来的通信域名，否则登录会报错
 ![Snipaste_2023-06-07_12-59-40](https://i2.100024.xyz/2023/06/07/ktvekz.webp)
 
 # 11. 常见坑点
 1. 配置agent时，将面板RPC端口（默认5555）错误地写成了NAT vps的可用端口
 2. Alpine手搓服务时，将面板RPC端口（默认5555）错误地写成了面板访问端口8008
 3. 套 CloudFlare 后提示“重定向的次数过多”，需要将SSL/TLS 设置，改为 完全（Full） 或 完全严格Full (strict) 模式
 
 ---
 最后祝大家都能搭建一个漂亮的探针界面，玩鸡快乐！ 
