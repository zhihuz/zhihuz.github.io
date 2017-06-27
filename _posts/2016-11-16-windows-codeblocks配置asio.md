---
layout: post
title: "windows codeblocks配置boost::asio"
tags: [c++ , asio,网络]
---

## 下载与安装
从[boost官网](http://www.boost.org/)下载最新版boost，本人下载的是boost\_1\_64_0.zip文件，解压缩。在文件根目录同时按下shift+鼠标右键，在此处打开命令窗口，执行 __cd tools\build\src\engine__ 命令，确认当前目录下存在build.bat文件，执行命令 __build gcc__,此目录下生成 bin.ntx86\bjam.exe和b2.exe 文件，将bjam.exe拷贝到源文件的根目录下。并在根目录下执行 __bjam --toolset=gcc --prefix=C:\boost install__，接下来便是漫长的等待。此命令会把编译后的boost文件安装到C:\boost下。

## 配置codeblocks
+ 打开cb，依次点击setting-global variables，点击new，输入新变量名称boost，参数如图所示（请根据实际版本修改路径)

![](https://raw.githubusercontent.com/zhihuz/BLOG-RES/master/2016-11-16%2034355.jpg)

+ 依次点击setting-compiler-search directories-Compiler，输入__$(#boost.include)__，如图所示

![](https://raw.githubusercontent.com/zhihuz/BLOG-RES/master/2016-11-16%2034356.jpg)

同理，点击Liner输入__$(#boost.lib)__

+ 点击setting-Linker settings，在link libraries中依次添加  
libboost\_filesystem-mgw63-mt-d-1\_64.a (C:\boost\lib\libboost\_filesystem-mgw63-mt-d-1\_64.a),  
libboost\_system-mgw63-mt-d-1\_64.a (C:\boost\lib\libboost\_system-mgw63-mt-d-1\_64.a),  
libwsock32.a (C:\MinGW\lib\libwsock32.a),  
libws2\_32.a (C:\MinGW\lib\libws2\_32.a)  
这四个文件，在other link options中添加 __-lwsock32__,如图所示

![](https://raw.githubusercontent.com/zhihuz/BLOG-RES/master/2016-11-16%2034357.jpg)

+ 依次点击setting-compiler-other compiler options，添加__\-lboost\_system__

## demo测试

##### client.cpp源码

{% highlight python linenos %}

#include <iostream>
#include "boost/asio.hpp"
#include <windows.h>

using namespace boost::asio;
using namespace std;

int main()
{
    boost::asio::io_service ioservice;
    boost::asio::ip::tcp::endpoint ep(boost::asio::ip::address::from_string("127.0.0.1"),2001);
    boost::asio::ip::tcp::socket sock(ioservice);
    sock.connect(ep);

    for(;;)
    {
      array<char, 512> buf  = {{  }};
      boost::system::error_code ignored_error;
      write(sock, boost::asio::buffer(buf), ignored_error);
      size_t len = sock.read_some(boost::asio::buffer(buf), ignored_error);
      cout.write(buf.data(), len);
      Sleep(2000);
    }
    return 0;
}

{% endhighlight %}

##### server.cpp源码

{% highlight python linenos %}

#include <iostream>
#include <boost/asio.hpp>

using namespace boost::asio;
using namespace std;

typedef boost::shared_ptr<ip::tcp::socket> socket_ptr;
io_service service;
ip::tcp::endpoint ep( ip::tcp::v4(), 2001);
ip::tcp::acceptor acc(service, ep);

void client_session(socket_ptr sock) {
    while ( true) {
        char data[512];
        size_t len = sock->read_some(buffer(data));
        if ( len > 0)
            write(*sock, buffer("ok\n", 3));
    }
}



int main(int argc,char * argv[]){
    while ( true) {
        socket_ptr sock(new ip::tcp::socket(service));
        acc.accept(*sock);
        detail::thread( std::bind(client_session, sock));
    }
}

{% endhighlight %}

编译执行即可（先运行server，再运行client）。若程序正常运行，则安装成功。若程序报错，则应优先检查以上配置步骤是否有遗漏。


