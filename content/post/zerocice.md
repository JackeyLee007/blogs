+++
Categories = ["架构", "zeroc ice"
]
Tags = ["rpc"
]
date = "2016-12-10T17:13:36+08:00"
title = "zerocice"

+++

用Python开发Zeroc-ICE应用
---

## 安装Zeroc-ICE
> sudo apt-key adv --keyserver keyserver.ubuntu.com --recv 5E6DA83306132997
sudo apt-add-repository "deb http://zeroc.com/download/apt/ubuntu$(lsb_release -rs) stable main"
sudo apt-get update
sudo apt-get install zeroc-ice-all-runtime zeroc-ice-all-dev


## 安装zeroc-ice的python开发包
> sudo -H pip install zeroc-ice

安装过程中可能出现缺少某些C/C++头文件的问题，例如缺少python.h、openssl/ssl.h、bzlib.h，这些都是因为没有安装相应的开发包。可以通过如下的命令解决：
> sudo apt-get install python-dev
> sudo apt-get install libssl-dev
> sudo apt-get install libbz2-dev

## 开发Server和Client
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
```
slice2py Printer.ice

```
&emsp;&emsp;其他语言可以依此类推，例如slice2java，slice2cpp。
&emsp;&emsp;命令执行成功，可以看到在目标目录中生成了一个Printer_ice.py文件，以及一个Demo目录。Demo是slice接口文件中定义的module名称。

### 编写服务器
``` pytho
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
    adapter = ic.createObjectAdapterWithEndpoints("SimplePrinterAdapter", "default -p 10000")
    object = PrinterI()
    adapter.add(object, ic.stringToIdentity("SimplePrinter"))
    adapter.activate()
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
server.cfg内容如下：
```
PrinterAdapter.AdapterId=PrinterAdapter
PrinterAdapter.Endpoints=tcp
```
### 编写客户端
``` python
import sys, traceback, Ice 
import Demo

status = 0 
ic = None

try:
    ic = Ice.initialize(sys.argv)
    base = ic.stringToProxy("SimplePrinter:default -p 10000")
    printer = Demo.PrinterPrx.checkedCast(base)
    if not printer:
        raise RuntimeError("Invalid proxy")

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
客户端的配置文件如下：
```
Ice.Default.Locator=SzcIceGrid/Locator:tcp -h 127.0.0.1 -p 4061
```
## 部署到icegrid
### 配置注册中心
registry.cfg
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
### 配置节点
node1.cfg
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
### 应用描述文件
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
