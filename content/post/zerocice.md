+++
Categories = ["架构", "平台"]
Tags = ["RPC", "Zeroc ICE"]
date = "2016-12-10T17:13:36+08:00"
title = "用Python开发Zeroc Ice应用"

+++

用Python开发Zeroc Ice应用
---
## Zeroc Ice简介
&emsp;&emsp;Zeroc ICE（Internet Communications Engine ，互联网通信引擎）是目前功能比较强大和完善的RPC框架，支持跨平台、跨语言调用。它非常灵活，可以通过TCP、UDP、SSL/TSL或WebSocket连接，支持同步、异步调用，以及服务器和客户端之间的双向连接。Zeroc ICE的效率非常高，它使用一种高效的二进制协议，对带宽的消耗比较小。甚至对于通过卫星的RPC调用，Zeroc ICE还可以对数据流进一步压缩。另外Zeroc ICE还可以在不解包的情况下转发调用请求，省去普通转发时的解包、重新压包的时间。 
&emsp;&emsp;Zeroc ICE的应用还可以部署在icegrid上，实现网格计算，即客户端调用时不必指定目标主机，由ICE负责查找；服务端也可以在调用时才开启，动态加载；同样的服务也可以部署多个，实现高可用。

## 实验简介
&emsp;&emsp;Zeroc ICE支持跨语言RPC调用，包括C++、C#、Java、JavaScript、Python、Objective-C、Ruby、PHP、VB等。本次实验采用Python（Pyhon 2.7以上，或者Python 3都可以）。实验的内容是在icegrid上部署一个简单的服务器，当客户端调用时输出指定内容，并返回一个字符串。实验步骤如下：

- 安装Zeroc ICE
- 开发服务端和客户端程序
- 部署到icegrid
- 客户端调用

## 环境准备
&emsp;&emsp; 本次实验采用的操作系统是Ubuntu 14.04。如果使用其他操作系统，可以根据[Zeroc ICE](https://zeroc.com/distributions/ice)的文档相应调整。
### 安装Zeroc Ice
&emsp;&emsp; 如果系统中没有安装Zeroc ICE，并且ubuntu的软件源中也没有zeroc ice，可以按照下面的步骤安装。
> sudo apt-key adv --keyserver keyserver.ubuntu.com --recv 5E6DA83306132997
> sudo apt-add-repository "deb http://zeroc.com/download/apt/ubuntu$(lsb_release -rs) stable main"
> sudo apt-get update
> sudo apt-get install zeroc-ice-all-runtime zeroc-ice-all-dev

&emsp;&emsp;安装之后系统中就有了slice2cpp、slice2java等Sliece（Zeroc ICE定义的接口描述语言，IDL）文件到相应语言的转换程序，以及icegrid、iceregistry、icegridadmin等程序。如果缺少目标语言的工具（例如slice2py）或开发包，还需要特别安装。

### 安装Zeroc ICE的python开发包
&emsp;&emsp;当然在这一步之前应当首先安装python和pip（python的依赖管理工具），此处略。Zeroc ICE的python开发包（或者模块）叫zeroc-ice，可以使用pip安装。
> sudo -H pip install zeroc-ice

&emsp;&emsp;安装过程中可能出现缺少某些C/C++头文件的问题，例如缺少python.h、openssl/ssl.h、bzlib.h，这些都是因为没有安装相应的开发包。可以通过如下的命令解决：
> sudo apt-get install python-dev
> sudo apt-get install libssl-dev
> sudo apt-get install libbz2-dev

## 开发Server和Client
&emsp;&emsp;下面即是真正的服务端和客户端开发。开发过程通常是:

- 使用Slice语言定义语言无关的接口文件
- 转换成指定语言的接口文件
- 根据接口文件开发服务端和客户端程序
- 书写服务端和客户端的配置文件

### 使用slice语言定义接口
```
// Printer.ice
module Demo {
	interface Printer {
	     string printString(string s);
	};
};
```

### 生成指定语言的接口文件
&emsp;&emsp;本次开发采用的语言是python，所以使用
> slice2py Printer.ice

&emsp;&emsp;其他语言可以依此类推，例如slice2java，slice2cpp。
&emsp;&emsp;命令执行成功，可以看到在目标目录中生成了一个Printer_ice.py文件，以及一个Demo目录。Demo是slice接口文件中定义的module名称。

### 编写服务器
``` python
import sys, traceback, Ice 
import Demo

# PrinterI是接口实现类，Demo.Printer是slice2py生成的接口
class PrinterI(Demo.Printer):
    def printString(self, s, current=None):
        print(s)
        return "Server Printed: " + s 

status = 0 
ic = None

try:
	# 初始化zeroc ice环境
    ic = Ice.initialize(sys.argv)
    # 生成名为SimplePrinterAdapter的对象适配器，连接方式是缺省的tcp，监听端口10000
    adapter = ic.createObjectAdapterWithEndpoints("SimplePrinterAdapter", "default -p 10000")
	# 生成接口的实现对象，并以指定的名字SimplePrinter添加到对象适配器中
    object = PrinterI()
    adapter.add(object, ic.stringToIdentity("SimplePrinter"))
	# 激活对象适配器
    adapter.activate()
    # 使得本服务器的调用线程在此暂停，直至ice服务结束，或者进程结束
    ic.waitForShutdown()
except:
    traceback.print_exc()
    status = 1 

if ic: 
    # Clean up
    try:
        ic.destroy()
    except:
        traceback.print_exc()
        status = 1 

sys.exit(status)
```
&emsp;&emsp;server.cfg内容如下：
```
PrinterAdapter.AdapterId=PrinterAdapter
PrinterAdapter.Endpoints=tcp
```
&emsp;&emsp;其中tcp的意思通过tcp协议调用，服务器监听来自tcp协议的连接请求。

### 编写客户端
``` python
import sys, traceback, Ice 
import Demo

status = 0 
ic = None

try:
    ic = Ice.initialize(sys.argv)
    # 生成名为SimplePrinter代理对象，且通过tcp调用，连接目标机器的10000端口
    base = ic.stringToProxy("SimplePrinter:default -p 10000")
    # 将代理对象转换成目标对象
    printer = Demo.PrinterPrx.checkedCast(base)
    if not printer:
        raise RuntimeError("Invalid proxy")
	# 调用服务器的printString方法，并输出返回结果
    rs = printer.printString("Hello World, I'm talking to you through RPC")
    print(rs)

except:
    traceback.print_exc()
    status = 1 

if ic: 
    # Clean up
    try:
        ic.destroy()
    except:
        traceback.print_exc()
        status = 1 

sys.exit(status)

```
&emsp;&emsp;客户端的配置文件如下：
```
Ice.Default.Locator=SzcIceGrid/Locator:tcp -h 127.0.0.1 -p 4061
```
&emsp;&emsp;其含义是使用本地（-h 127.0.0.1）上的名为SzcIceGrid的服务定位器，连接协议是tcp，端口4061。**注意**：与注册中心的配置相对应。
### 客户端直连服务端
&emsp;&emsp;上述程序开发完毕之后不用部署到icegrid就可以直接运行，配置文件是用来在icegrid上定位和连接服务。此时可以一边运行服务端，一边运行客户端，检验一下它们的功能。
> python Server.py

&emsp;&emsp;运行之后可以看到进程并没有结束，一直在等待连接。然后另起一个终端，运行客户端程序。
> python Client.py

&emsp;&emsp;运行之后可以看到服务端和客户端窗口的输出。
![](http://ohpg9orwb.bkt.clouddn.com/blogs/icegrid/client_to_server.PNG)
## 部署到icegrid
&emsp;&emsp;icegrid是Zeroc ICE的云计算解决方案。它可以将各种服务端部署在多台机器上，并为客户端调用提供服务定位、服务激活、负载均衡、故障转移等服务。客户端只要连接到指定的服务注册中心，就可以根据服务名称（这里是SimplePrinter），以及连接协议（Endpoints，这里是tcp）就可以找到相应的服务。服务在注册时也不必处在运行状态，可以由icegrid根据调用请求，自动启动。
![](https://doc.zeroc.com/download/attachments/16716477/Simple_IceGrid_application.gif?version=1&modificationDate=1432840085000&api=v2)
### 配置注册中心
registry.cfg（服务注册中心的配置文件）
```
IceGrid.InstanceName=SzcIceGrid  
#客户端连接到注册中心的地址  
IceGrid.Registry.Client.Endpoints=tcp   -p 4061
IceGrid.Registry.Server.Endpoints=tcp
IceGrid.Registry.Internal.Endpoints=tcp
IceGrid.Registry.PermissionsVerifier=SzcIceGrid/NullPermissionsVerifier
IceGrid.Registry.AdminPermissionsVerifier=SzcIceGrid/NullPermissionsVerifier
#注册中心数据保存路径,需要手动创建文件夹
IceGrid.Registry.Data=/home/rocway/test/zerocice/registry
IceGrid.Registry.DynamicRegistration=1
Ice.Admin.InstanceName=AdminInstance
Ice.Admin.ServerId=Admin
```
**注意**：手工创建文件中的路径。
### 配置节点
&emsp;&emsp;节点是服务所在的机器。在实际生产环境中，服务注册中心也可以运行在其中某个节点上。
node1.cfg（服务所在节点的配置文件）
```
# 注册中心地址  
Ice.Default.Locator=SzcIceGrid/Locator:tcp -h 127.0.0.1 -p 4061  
#node名  
IceGrid.Node.Name=node1  
IceGrid.Node.Endpoints=tcp  
#node存储路径  
IceGrid.Node.Data=/home/rocway/test/zerocice/nodes/node1
IceGrid.Node.Output=/home/rocway/test/zerocice/nodes/node1
IceGrid.Node.CollocateRegistry=0
```
**注意**：手工创建上述文件中提到的路径。其中服务端程序的输出会保存在Ouput指向路径的*.out文件中。
### 应用描述文件
&emsp;&emsp;应用描述文件用来描述服务端程序在icegrid中的部署情况。包括应用的名称、服务程序的路径、执行参数等等。
app.xml
``` xml
<icegrid>
    <application name="PrinterApplication">
        <node name="node1">
            <server id="PrinterServer" exe="python" activation="on-demand">
                <adapter name="PrinterAdapter" endpoints="tcp -h 127.0.0.1">
                    <object identity="SimplePrinter" type="::Demo::Printer" property="Identity"/>
                </adapter>
                <option>/home/rocway/test/zerocice/Server.py</option>   
                <property name="Ice.Trace.Network" value="1"/>
                <properties>  
                    <property name="Ice.ThreadPool.Server.SizeMax" value="1" />  
                </properties>  
                        <property name="IceMX.Metrics.Debug.GroupBy" value="id"/>
                        <property name="IceMX.Metrics.Debug.Disabled" value="1"/>
                        <property name="IceMX.Metrics.ByParent.GroupBy" value="parent"/>
                        <property name="IceMX.Metrics.ByParent.Disabled" value="1"/>     
            </server>
        </node>
    </application>
</icegrid>

```

### 启动icegrid

1. 启动icegrid注册中心
 icegridregistry --Ice.Config=registry.cfg
![](http://ohpg9orwb.bkt.clouddn.com/blogs/icegrid/grid_start_registry.PNG)
2. 启动某个节点
icegridnode --Ice.Config=node1.cfg
![](http://ohpg9orwb.bkt.clouddn.com/blogs/icegrid/grid_start_node.PNG)
3. 启动节点上的应用管理程序, 并添加应用
 icegridadmin --Ice.Config=node1.cfg
 application add app.xml
 ![](http://ohpg9orwb.bkt.clouddn.com/blogs/icegrid/grid_admin_node.PNG)
4. 查看已经添加的应用
application describe PrinterApplication
![](http://ohpg9orwb.bkt.clouddn.com/blogs/icegrid/grid_describe_application.PNG)
5. 启动各节点上的应用服务
icegridgui
![](http://ohpg9orwb.bkt.clouddn.com/blogs/icegrid/connect_grid.PNG)
![](http://ohpg9orwb.bkt.clouddn.com/blogs/icegrid/grid_start_server.PNG)
6. 运行客户端程序
python Client.py
![](http://ohpg9orwb.bkt.clouddn.com/blogs/icegrid/grid_ClientRun_Server_Response.PNG)
 ```
## 实验总结

&emsp;&emsp;此次实验实现了在icegrid上部署服务程序，客户端通过icegrid的服务注册中心调用该服务。实验中服务端和客户端使用的都是Python，有兴趣的同学也可以分别使用不同的语言开发服务端和客户端，尝试一下Zeroc ICE的跨语言RPC调用。
&emsp;&emsp;本次实验就到这里，有关Zeroc ICE的其他内容请关注后续的课程。

## 版权声明

&emsp;&emsp;此课程内容由作者RocWay提供并授权使用，实验楼基于原著进行了内容和章节的优化，修正了一些错误。版权归原作者所有。未经允许，不得以任何形式进行传播和发布
