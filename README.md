# janus-webrtc-server-setup
记录个人搭建 Janus WebRTC 服务器过程的笔记与总结，包含遇到的问题和解决方案。

------------------------------------------------------

**开始之前**：

```bash
sudo apt install git aptitude
```



## 安装依赖库

```bash
sudo aptitude install libmicrohttpd-dev libjansson-dev \
	libssl-dev libsrtp-dev libsofia-sip-ua-dev libglib2.0-dev \
	libopus-dev libogg-dev libcurl4-openssl-dev liblua5.3-dev \
	libconfig-dev pkg-config gengetopt libtool automake
```



## 安装libnice

直接从 `https://launchpad.net/ubuntu/+source/libnice/0.1.16-1` 这个地址下载老版本

下载libnice_0.1.16.orig.tar.gz，解压后执行

```bash
tar xfv libnice_0.1.16.orig.tar.gz
./configure && make && sudo make install
```



## 安装libsrtp

```bash
wget https://github.com/cisco/libsrtp/archive/v2.2.0.tar.gz
tar xfv v2.2.0.tar.gz
cd libsrtp-2.2.0
./configure --prefix=/usr --enable-openssl
make shared_library && sudo make install
```



## 安装libwebsocket

```bash
git clone https://github.com/warmcat/libwebsockets.git
cd libwebsockets
mkdir build
cd build
cmake -DLWS_MAX_SMP=1 -DCMAKE_INSTALL_PREFIX:PATH=/usr -DCMAKE_C_FLAGS="-fpic" ..
make && sudo make install
```

之后会安装至 `/usr/include/libwebsockets` 路径下



## 安装janus

```bash
git clone https://github.com/meetecho/janus-gateway.git
cd janus-gateway
sh autogen.sh
./configure --prefix=/opt/janus
make
sudo make install
sudo make configs
```



configure信息：

```bash
Compiler:                  gcc
libsrtp version:           2.x
SSL/crypto library:        OpenSSL
DTLS set-timeout:          not available
Mutex implementation:      GMutex (native futex on Linux)
DataChannels support:      no
Recordings post-processor: no
TURN REST API client:      yes
Doxygen documentation:     no
Transports:
    REST (HTTP/HTTPS):     yes
    WebSockets:            yes
    RabbitMQ:              no
    MQTT:                  no
    Unix Sockets:          yes
    Nanomsg:               no
Plugins:
    Echo Test:             yes
    Streaming:             yes
    Video Call:            yes
    SIP Gateway:           no
    NoSIP (RTP Bridge):    yes
    Audio Bridge:          no
    Video Room:            yes
    Record&Play:           yes
    Text Room:             yes
    Lua Interpreter:       no
    Duktape Interpreter:   no
Event handlers:
    Sample event handler:  yes
    WebSocket ev. handler: yes
    RabbitMQ event handler:no
    MQTT event handler:    no
    Nanomsg event handler: no
    GELF event handler:    yes
External loggers:
    JSON file logger:      no
JavaScript modules:        no
```

`REST (HTTP/HTTPS)` 和 `WebSockets` 以及所要测试插件都要是yes



> 如果遇到了 `REST (HTTP/HTTPS)` 为 `no` ，需要安装 libmicrohttpd-dev 包：
>
> ```bash
> sudo apt install libmicrohttpd-dev
> ```
>
> 然后重新运行 configure
>
> ```bash
> ./configure --prefix=/opt/janus
> ```





## 配置HTTPS

- 安装Nginx

  ```bash
  sudo apt install nginx
  ```

  

- 生成密钥

  ```bash
  cd /opt/janus/share/janus
  mkdir cert
  cd cert
  
  sudo openssl req -x509 -newkey rsa:2048 -keyout mycer.key -out mycer.pem -days 99999 -nodes
  ```

  

- 添加https服务

  在 `/etc/nginx/conf.d/default.conf` 中修改关键信息

  ```bash
  sudo vim /etc/nginx/conf.d/default.conf
  ```

  ```
  server {
  	listen 443 ssl;
      ssl_certificate       /opt/janus/share/janus/cert/mycer.pem; #公钥完整路径
      ssl_certificate_key       /opt/janus/share/janus/cert/mycer.key; #私钥完整路径
  
      location / {
          root   /opt/janus/share/janus/html;
      }
  
  }
  ```

  



- 修改janus信息

  在 `/opt/janus/etc/janus/` 路径下修改 `janus.jcfg` 文件为如下的格式

  ```bash
  sudo vim /opt/janus/etc/janus/janus.jcfg
  ```

  ```ini
  certificates: {
  	cert_pem = "/opt/janus/share/janus/cert/mycer.pem"
  	cert_key = "/opt/janus/share/janus/cert/mycer.key"
  	#cert_pwd = "secretpassphrase"
  	#dtls_accept_selfsigned = false
  	#dtls_ciphers = "your-desired-openssl-ciphers"
  	#rsa_private_key = false
  }
  ```

  然后在当前路径下打开 `janus.transport.http.jcfg` 文件

  ```bash
  sudo vim /opt/janus/etc/janus/janus.transport.http.jcfg
  ```

  把 `https = false` 修改为true；去掉 `cert_pem` 和 `cert_key` 的注释然后修改为实际路径

  ```ini
  general: {
  	#events = true					# Whether to notify event handlers about transport events (default=true)
  	json = "indented"				# Whether the JSON messages should be indented (default),
  									# plain (no indentation) or compact (no indentation and no spaces)
  	base_path = "/janus"			# Base path to bind to in the web server (plain HTTP only)
  	http = true						# Whether to enable the plain HTTP interface
  	port = 8088						# Web server HTTP port
  
  	https = true					# 修改为true
  
  }
  
  
  certificates: {
  	cert_pem = "/opt/janus/share/janus/cert/mycer.pem"
  	cert_key = "/opt/janus/share/janus/cert/mycer.key"
  	#cert_pwd = "secretpassphrase"
  	#ciphers = "PFS:-VERS-TLS1.0:-VERS-TLS1.1:-3DES-CBC:-ARCFOUR-128"
  }
  ```

  

  



## 启动janus

janus可执行程序在 `/opt/janus/bin/` 路径下可通过 `/opt/janus/bin/janus` 直接启动。启动后的显示的信息如下：

```bash
Logger plugins folder: /opt/janus/lib/janus/loggers
[WARN]  Couldn't access logger plugins folder...
---------------------------------------------------
  Starting Meetecho Janus (WebRTC Server) v1.3.2
---------------------------------------------------

Checking command line arguments...
Debug/log level is 4
Debug/log timestamps are disabled
Debug/log colors are enabled
Adding 'vmnet' to the ICE ignore list...
Using 10.15.15.141 as local IP...
Token based authentication disabled
Initializing recorder code
Initializing ICE stuff (Full mode, ICE-TCP candidates disabled, half-trickle, IPv6 support disabled)
TURN REST API backend: (disabled)
[WARN] Janus is deployed on a private address (10.15.15.141) but you didn't specify any STUN server! Expect trouble if this is supposed to work over the internet and not just in a LAN...
Crypto: OpenSSL >= 1.1.0
Fingerprint of our certificate: 5B:91:9C:88:53:00:64:12:3F:87:20:6A:E3:8D:C5:5B:7B:98:7E:8E:0E:B6:8E:C4:51:30:90:59:70:F5:40:E9
[WARN] Data Channels support not compiled
Sessions watchdog started
Event handlers support disabled
Plugins folder: /opt/janus/lib/janus/plugins
Loading plugin 'libjanus_streaming.so'...
Joining Janus requests handler thread
[WARN] libogg not available, Streaming plugin will not have file-based Opus streaming
JANUS Streaming plugin initialized!
Loading plugin 'libjanus_videocall.so'...
JANUS VideoCall plugin initialized!
Loading plugin 'libjanus_textroom.so'...
[WARN] Data channels support not compiled, disabling TextRoom plugin
[WARN] The 'janus.plugin.textroom' plugin could not be initialized
Loading plugin 'libjanus_echotest.so'...
JANUS EchoTest plugin initialized!
Loading plugin 'libjanus_videoroom.so'...
JANUS VideoRoom plugin initialized!
Loading plugin 'libjanus_recordplay.so'...
JANUS Record&Play plugin initialized!
Loading plugin 'libjanus_nosip.so'...
JANUS NoSIP plugin initialized!
Transport plugins folder: /opt/janus/lib/janus/transports
Loading transport plugin 'libjanus_websockets.so'...
[WARN] libwebsockets has been built without IPv6 support, will bind to IPv4 only
libwebsockets logging: 0
Websockets server started (port 8188)...
JANUS WebSockets transport plugin initialized!
Loading transport plugin 'libjanus_http.so'...
WebSockets thread started
HTTP transport timer started
HTTP webserver started (port 8088, /janus path listener)...
HTTPS webserver started (port 8089, /janus path listener)...
JANUS REST (HTTP/HTTPS) transport plugin initialized!
Loading transport plugin 'libjanus_pfunix.so'...
[WARN] No Unix Sockets server started, giving up...
[WARN] The 'janus.transport.pfunix' plugin could not be initialized
```

需要确保以上的插件都显示 `plugin initialized` ，以及 `Websockets` 相关和 `REST (HTTP/HTTPS)` 都启动成功，否则Demo中的功能可能无法正常使用。启动不成功的话可能需要检查自己的依赖，重新进行编译







## 运行Demo

前面我们已经配置好了Nginx和janus的相关配置，现在只需重启Nginx即可

```bash
sudo systemctl restart nginx.service
```



需要注意的是，如果跳过了Https服务配置阶段的操作，那么在谷歌浏览器中是无法正常运行 `Video Room` 的demo的，因为谷歌浏览器会禁止在http下拉起摄像头



通过 `https://192.168.x.x` 运行Demo，在Video Room中可以就可看到自己的摄像头画面了。



> 以上的操作搭建起来的janus服务器只限于在局域网内运行。如果想将WebRTC服务器搭建在公网环境中，那么还需要配置STUN服务器。这部分我暂时还未研究，后续再更新

