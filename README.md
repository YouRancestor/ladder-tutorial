# 云梯搭设指南

## 工具

- 一台远(jing)程(wai)虚拟主机。

- ssh。

- [privoxy](https://www.privoxy.org) / [polipo](https://github.com/jech/polipo.git)。（可选）

- [proxifier](https://proxifier.soft32.com) / [proxycap](http://www.proxycap.com) / [proxychains](https://github.com/haad/proxychains)([-windows](https://github.com/shunf4/proxychains-windows.git))。（可选）

## 方法一： PPTP / L2TP 服务器搭设

参考链接 [Rolling out your own private VPN server on AWS cloud in 10 minutes](https://github.com/webdigi/AWS-VPN-Server-Setup)

以上教程使用脚本（CloudFormation堆栈）自动创建亚马逊ec2实例，并根据预设参数安装启动PPTP或L2TP服务器程序。脚本不含ssh服务，所以不能用于远程连接。也可手动创建ec2实例，并参考脚本自行搭设所需的服务程序，这样可以与后面的方法二并存。

优点：操作系统通常都支持PPTP和L2TP协议进行全局代理。

缺点：客户端通常由系统内置，访问服务器的目标端口通常是固定的不易修改，因此数据包容易被识别并拦截；代理为全局代理，不够灵活。

## 方法二： socks5 代理服务搭设

假设你已经配置好一个远程主机，并可以通过ssh远程登陆。（如果你没有主机，可以参考方法一中的链接申请一个aws的账号，但是不要使用脚本，只需手动创建一个ec2实例即可。）

本地如果是Windows环境，推荐使用git-bash作为终端。

1. 使用 ssh tunnel 建立socks5服务

    ```sh
    ssh -TND socks5_port -i /path/to/private_key user_name@server_address
    ```

    该命令在使用密钥private_key以user_name登陆地址为server_address主机的同时，在本地（127.0.0.1）的socks5_port端口建立一个socks5代理服务。至此，使用支持socks5代理的浏览器（如火狐）并将代理配置为上述socks5服务后，即可访问境外网络。

1. socks5 代理转 http 代理

    若使用的浏览器不支持socks5代理，可以使用工具将socks5代理转为http代理，这里使用polipo为例。

    ```sh
    polipo socksParentProxy=127.0.0.1:socks5_port proxyAddress=0.0.0.0 proxyPort=http_port
    ```

    该命令将本地socks5_port端口的socks5代理转换为http代理，http代理的端口号为http_port。之后即可为浏览器配置http代理或将系统代理配置为127.0.0.1:http_port。proxyAddress=0.0.0.0中的0.0.0.0的意思是代理服务在本机所有可用的IP地址上监听，以接受来自本机以及局域网内其他设备的请求，可为局域网内的其他设备提供代理服务（注意配置防火墙放行端口）；如仅希望本机访问，也可替换为proxyAddress=127.0.0.1。

    注：

    polipo需要自行编译，下载源码，在源码目录下用gcc/mingw作为编译器直接make很容易编过。

    Windows 环境下 Makefile 文件需适当修改，参考：

    找到Makefile中以下几行

    ```Makefile
    # CC = gcc

    # EXE=.exe
    # LDLIBS = -lws2_32
    ```

    将井号去掉，改为

    ```Makefile
    CC = gcc

    EXE=.exe
    LDLIBS = -lws2_32
    ```

    即可。

    另外，Makefile中定义的PREFIX，LOCAL_ROOT，DISK_CACHE_ROOT几个变量用于指定安装位置和运行时缓存目录，可改可不改，不影响程序运行。如有需要自行修改。

1. 其他应用

    如有其他不支持http或socks5代理的应用需要通过代理访问网络，可以使用 proxifier, proxycap, proxychains 等软件将应用的数据包截获并转发至代理服务。使用方法请自行搜索或查阅文档。

优点：服务端容易（几乎无需）配置，只需ssh服务（通常默认开启）即可；转发的数据包通过ssh信道进行加密传输，而ssh为远程访问的通用软件，不易被查封，还可以更换端口；灵活控制，可以为每一个应用分别指定使用或不使用代理。

缺点：如需全局全协议代理，客户端需要的工具较多，配置较为繁琐。

