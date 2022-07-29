# Linux进程间通信：dbus的使用—— D-Bus介绍及signal、method测试例程

# 总体介绍

#### D-Bus的三个层面

D-Bus是一个为应用程序间通信的消息总线系统, 用于进程之间的通信。它是个3层[架构](https://so.csdn.net/so/search?q=架构&spm=1001.2101.3001.7020)的IPC 系统，包括：

- 函数库libdbus，用于两个应用程序互相联系和交互消息。
- 一个基于libdbus构造的消息总线守护进程，可同时与多个应用程序相连，并能把来自一个应用程序的消息路由到0或者多个其他程序。
- 基于特定应用程序框架的封装库或捆绑（wrapper libraries or bindings）。例如，libdbus-glib和libdbus-qt，还有绑定在其他语言，例如Python的。大多数开发者都是使用这些封装库的API，因为它们简化了D-Bus编程细节。libdbus被有意设计成为更高层次绑定的底层后端（low-level backend）。大部分libdbus的 API仅仅是为了用来实现绑定。

#### System bus和session bus

  在D-Bus中，“bus”是核心的概念，它是一个通道：不同的程序可以通过这个通道做些操作，比如方法调用、发送信号和监听特定的信号。在一台机器上总线守护有多个实例(instance)。这些总线之间都是相互独立的。

一个持久的*系统总线（system bus）：*

- 它在引导时就会启动。这个总线由操作系统和后台进程使用，安全性非常好，以使得任意的应用程序不能欺骗系统事件。
- 它是桌面会话和操作系统的通信，这里操作系统一般而言包括内核和系统守护进程。
- 这种通道的最常用的方面就是发送系统消息，比如：插入一个新的存储设备；有新的网络连接；等等。

还将有很多会话总线（session buses）：

- 这些总线当用户登录后启动，属于那个用户私有。它是用户的应用程序用来通信的一个会话总线。
- 同一个桌面会话中两个桌面应用程序的通信，可使得桌面会话作为整体集成在一起以解决进程生命周期的相关问题。 这在GNOME和KDE桌面中大量使用。

  对于一些远程的工作站，在system bus中可能会有一些问题，例如热插拔，是否需要通知远端的terminal，这会是的kernel暴露一些设备的能力，不过，我们现在关心D-Bus，是因为手持终端设备的使用，这些将不会出现。在Internet Tablet，也包括我们的手机系统，所有的应用程序都是使用一个用户ID运行的，所以只有一个会话通道，这一点是和Linux桌面系统是有明显区别的。

  D-Bus是低延迟而且低开销的，设计得小而高效，以便最小化传送的往返时间。另外，协议是二进制的，而不是文本的，这样就排除了费时的序列化过程。从开发者的角度来看，D-BUS 是易于使用的。有线协议容易理解，客户机程序库以直观的方式对其进行包装。D-Bus的主要目的是提供如下的一些更高层的功能：

- 结构化的名字空间
- 独立于架构的数据格式
- 支持消息中的大部分通用数据元素
- 带有异常处理的通用远程调用接口
- 支持广播类型的通信

#### Bus daemon总线守护

  Bus daemon是一个特殊的进程：这个进程可以从一个进程传递消息给另外一个进程。当然了，如果有很多applications链接到这个通道上，这个 daemon进程就会把消息转发给这些链接的所有程序。在最底层，D-Bus只支持点对点的通信，一般使用本地套接字（AF_UNIX）在应用和bus daemon之间通信。D-Bus的点对点是经过bus daemon抽象过的，由bus daemon来完成寻址和发送消息，因此每个应用不必要关心要把消息发给哪个进程。D-Bus发送消息通常包含如下步骤（正常情况下）：

- 创建和发送消息 给后台bus daemon进程，这个过程中会有两个上下文的切换
- 后台bus daemon进程会处理该消息，并转发给目标进程，这也会引起上下文的切换
- 目标程序接收到消息，然后根据消息的种类，做不同的响应：要么给个确认、要么应答、还有就是忽略它。最后一种情况对于“通知”类型的消息而言，前两种都会引起进一步的上 下文切换。

  综上原因，如果你准备在不同的进程之间传递大量的 数据，D-Bus可能不是最有效的方法，最有效的方法是使用共享内存，但是对共享内存的管理也是相当复杂的。



# 基本概念

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210417232448361.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDQ5ODMxOA==,size_16,color_FFFFFF,t_70#pic_center)

##### 原生对象和对象路径

  所有使用D-BUS的应用程序都包含一些对象, 当经由一个D-BUS连接受到一条消息时，该消息是被发往一个对象而不是整个应用程序。在开发中程序[框架](https://so.csdn.net/so/search?q=框架&spm=1001.2101.3001.7020)定义着这样的对象，例如JAVA，GObject，QObject等等，在D-Bus中成为native object。

  对于底层的D-Bus协议，即libdbus API，并不理会这些native object，它们使用的是一个叫做object path的概念。通过object path，高层编程可以为对象实例进行命名，并允许远程应用引用它们。这些名字看起来像是文件系统路径，例如一个对象可能叫做“/org/kde/kspread/sheets/3/cells/4/5”。 易读的路径名是受鼓励的做法，但也允许使用诸如“/com/mycompany/c5yo817y0c1y1c5b”等，只要它可以为你的应用程序所用。Namespacing的对象路径以开发者所有的域名开始（如 /org/kde）以避免系统不同代码模块互相干扰。

> 简单地说：一个应用创建对象实例进行D-Bus的通信，这些对象实例都有一个名字，命名方式类似于路径，例如/com/mycompany，这个名字在全局（session或者system）是唯一的，用于消息的路由。

##### 方法和信号Methods and Signals

  每一个对象有两类成员：方法和信号。方法就是JAVA中同样概念，方法是一段函数代码，带有输入和输出。信号是广播给所有兴趣的其他实体，信号可以带有数据payload。

  Tutorial这里说法有点不太清楚。在 D-BUS 中有四种类型的消息：方法调用（method calls）、方法返回（method returns）、信号（signals）和错误（errors）。要执行 D-BUS 对象的方法，您需要向对象发送一个方法调用消息。它将完成一些处理（就是执行了对象中的Method，Method是可以带有输入参数的。）并返回，返回消息或者错误消息。信号的不同之处在于它们不返回任何内容：既没有“信号返回”消息，也没有任何类型的错误消息。

##### 接口Interface

  每一个对象支持一个或者多个接口，接口是一组方法和信号，接口定义一个对象实体的类型。D-Bus对接口的命名方式，类似org.freedesktop.Introspectable。开发人员通常将使用编程语言类的的名字作为接口名字。

##### Proxies代理

  代理对象用来表示其他远程的remote object。当我们触发了proxy对象的method时，将会在D-Bus上发送一个method_call的消息，并等待答复，根据答复返回。使用非常方便，就像调用一个本地的对象。

上面是从开发人员编程的角度，经常涉及的一些概念，下面是D-Bus在工作或者处理时所涉及的一些概念。

##### Bus Names总线名字

  当一个应用连接到bus daemon，daemon立即会分配一个名字给这个连接，称为unique connection name ，这个唯一标识的名字以冒号：开头，例如:34-907，这个名字在daemon的整个生命周期是唯一的。但是这种名字总是临时分配，无法确定的，也难以记忆，因此应用可以要求有另外一个名字well-known name 来对应这个唯一标识，就像我们使用域名来对应IP地址一样。例如可以使用com.mycompany来映射:34-907。应用程序可能会要求拥有额外的周知名字（well-known name ） 。例如，你可以写一个规范来定义一个名字叫做 com.mycompany.TextEditor。你的协议可以指定自己拥有这个名字，一个应用程序应该在路径/com/mycompany /TextFileManager下有一个支持接口org.freedesktop.FileHandler的对象。应用程序就可以发送消息到这个总线名字，对象，和接口以执行方法调用。

  当一个应用结束或者崩溃是，OS kernel会关闭它的总线连接。总线发送notification消息告诉其他应用，这个应用的名字已经失去他的owner。当检测到这类notification时，应用可以知道其他应用的生命周期。这种方式也可用于只有一个实例的应用，即不开启同样的两个应用的情况。

##### 地址

  连接建立有server和client，对于bus daemon，应用就是client，daemon是server。一个D-Bus的地址是指server用于监听，client用于连接的地方，例如unix:path=/tmp/abcedf标识server将在路径/tmp/abcedf的UNIX domain socket监听。地址可以是指定的TCP/IP socket或者其他在或者将在D-Bus协议中定义的传输方式。

  如果使用bus daemon，libdbus将通过读取环境变量自动获取session bus damon的地址，通过检查一个指定的UNIX domain socket路径获取system bus的地址。如果使用D-bus，但不是daemon，需要定义那个应用是server，那个是client，并定义一套机制是他们认可server的地址，这不是通常的做法。

  通过上面的描述，我们可以获得下面的视图：

> **Address –> [Bus Name] –> Path –> Interface –> Method**
>
> bus name不是必要的，它只在daemon的情况下用于路由，点对点的直接连接是不需要的。
>
> 简单地说 ：Address是D－Bus中server用来监听client的地址，当一个client连接上D-Bus，通常是Daemo的方式，这个client就有了一个Bus Name。其他应用可以根据消息中所带的Bus Name，来判断和哪个应用相关。消息在总线中传递的时候，传递到应用中，再根据object path，送至应用中具体的对象实例中，也就是是应用中根据Interface创建的对象。这些Interface有method和singal两种，用来发送、接收、响应消息。

  这些概念对初学者可能会有些混淆，但是通过后面学习一个小程序，就很清楚，这是后面一个例子的示意图，回过头来看看之前写的这篇文章，这个示意图或许会更清楚。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210417232543564.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDQ5ODMxOA==,size_16,color_FFFFFF,t_70#pic_center)

# 消息

消息通过D-Bus在进程间传递。有四类消息：

> 一、Method call消息：将触发对象的一个method
> 二、Method return消息：触发的方法返回的结果
> 三、Error消息：触发的方法返回一个异常
> 四、Signal消息：通知，可以看作为事件消息。

  一个消息有消息头header，里面有field，有一个消息体body，里面有参数arguments。消息头包含消息体的路由信息，消息体就是净荷payload。头字段可能包括发送者的bus名，目的地的bus名，方法或者signal名等等，其中一个头字段是用于描述body中的参数的类型，例如“i”标识32位整数，"ii”表示净荷为2个32为整数。

##### 发送Method call消息的场景

  一个method call消息从进程A到进程B，B将应答一个method return消息或者error消息。在每个call消息带有一个序列号，应答消息也包含同样的号码，使之可以对应起来。他们的处理过程如下：

- 如果提供proxy，通过触发本地一个对象的方法从而触发另一个进程的远端对象的方法。应用调用proxy的一个方法，proxy构造一个method call消息发送到远端进程。
- 对于底层的API，不使用proxy，应用需要自己构造method call消息。
- 一个method call消息包含：远端进程的bus name，方法名字，方法的参数，远端进程中object path，可选的接口名字。
- method call消息发送到bus daemon
- bus daemon查看目的地的bus name。如果一个进程对应这个名字，bus daemon将method call消息发送到该进程中。如果没有发现匹配，bus daemon创建一个error消息作为应答返回。
- 进程接收后将method call消息分拆。对于简单的底层API情况，将立即执行方法，并发送一个method reply消息给bus daemon。对于高层的API，将检查对象path，interface和method，触发一个native object的方法，并将返回值封装在一个method reply消息中。
- bus daemon收到method reply消息，将其转发到原来的进程中
- 进程查看method reply消息，获取返回值。这个响应也可以标识一个error的残生。当使用高级的捆绑，method reply消息将转换为proxy方法的返回值或者一个exception。

  Bus daemon保证message的顺序，不会乱序。例如我们发送两个method call消息到同一个接受方，他们将按顺序接受。接收方并不要求一定按顺序回复。消息有一个序列号了匹配收发消息。

##### 发送Signal的场景

  signal是个广播的消息，不需要响应，接收方向daemon注册匹配的条件，包括发送方和信号名，bus守护只将信号发送给希望接受的进程。处理流程如下：

- 一个signal消息发送到bus daemon。
- signal消息包含发布该信号的interface名字，signal的名字，进程的bus名字，以及参数。
- 任何进程都可以注册的匹配条件（match rules)表明它所感兴趣的signal。总线有个注册match rules列表。
- bus daemon检查那些进程对该信号有兴趣，将信号消息发送到这些进程中。
- 收到信号的进程决定如何处理。如果使用高层的捆绑，一个porxy对象将会十分一个native的信号。如果使用底层的API，进程需要检查信号的发送发和信号的名字决定如果进行处理。

##### Introspection

  D-Bus对象可能支持一个接口org.freedesktop.DBus.Introspectable。该接口有一个方法Introspect，不带参数，将返回一个XML string。这个XML字符串描述接口，方法，信号。

  D-Bus提供两个命令dbus-monitor，可以查看bus，dbus-send命令，可以发送消息，可以用man来检查：

> dbus-send [–system | --session] --type=method_call（或者是signal,缺省是signal） --print-reply --dest=连接名 对象路径 接口名.方法名 参数类型:参数值 参数类型:参数值

我们通过这个接口.方法来更好地了解Ｄ-Bus。我们使用到一个方法ListNames来查看：

```powershell
[wei@wei-desktop ~]$ dbus-send --print-reply --type=method_call --dest=org.freedesktop.DBus / org.freedesktop.DBus.ListNames
method return sender=org.freedesktop.DBus -> dest=:1.75 reply_serial=2
   array [
      string "org.freedesktop.DBus"
      string ":1.7"
      string "org.freedesktop.Notifications "
      string "org.freedesktop.Telepathy.Client.EmpathyMoreThanMeetsTheEye"
      ... ...
      string ":1.6"
      string ":1.19"
   ]

# 例如其中有org.freedesktop.Notifications这样一个Name，我们希望进一步查看

[wei@wei-desktop ~]$ dbus-send --print-reply --type=method_call --dest=org.freedesktop.Notifications / org.freedesktop.DBus.Introspectable.Introspect
method return sender=:1.19 -> dest=:1.79 reply_serial=2
   string "<!DOCTYPE node PUBLIC "-//freedesktop//DTD D-BUS Object Introspection 1.0//EN"
"http://www.freedesktop.org/standards/dbus/1.0/introspect.dtd">
<node>
  <node name="org"/>
</node>
"

# 例如Node名字，表明路径，从/org开始

[wei@wei-desktop ~]$ dbus-send --print-reply --type=method_call --dest=org.freedesktop.Notifications /org org.freedesktop.DBus.Introspectable.Introspect
method return sender=:1.19 -> dest=:1.80 reply_serial=2
   string "<!DOCTYPE node PUBLIC "-//freedesktop//DTD D-BUS Object Introspection 1.0//EN"
"http://www.freedesktop.org/standards/dbus/1.0/introspect.dtd">
<node>
  <node name="freedesktop"/>
  <node name="moblin"/>
</node>
"

# Node名字，表明路径，有从/org/freedesktop开始
[wei@wei-desktop ~]$ dbus-send --print-reply --type=method_call --dest=org.freedesktop.Notifications /org/freedesktop org.freedesktop.DBus.Introspectable.Introspect
method return sender=:1.19 -> dest=:1.81 reply_serial=2
   string "<!DOCTYPE node PUBLIC "-//freedesktop//DTD D-BUS Object Introspection 1.0//EN"
"http://www.freedesktop.org/standards/dbus/1.0/introspect.dtd">
<node>
  <node name="Notifications"/>
</node>

"

# Node名字，表明路径，有从/org/freedesktop/Notifications开始，可以获取这个接口有什么method和singnal。
[wei@wei-desktop ~]$ dbus-send --print-reply --type=method_call --dest=org.freedesktop.Notifications /org/freedesktop/Notifications org.freedesktop.DBus.Introspectable.Introspect
method return sender=:1.19 -> dest=:1.82 reply_serial=2
   string "<!DOCTYPE node PUBLIC "-//freedesktop//DTD D-BUS Object Introspection 1.0//EN"
"http://www.freedesktop.org/standards/dbus/1.0/introspect.dtd">
<node>
  <interface name="org.freedesktop.DBus.Introspectable">
    <method name="Introspect">
      <arg name="data" direction="out" type="s"/>
    </method>
  </interface>
  <interface name="org.freedesktop.DBus.Properties">
    <method name="Get">
      <arg name="interface" direction="in" type="s"/>
      <arg name="propname" direction="in" type="s"/>
      <arg name="value" direction="out" type="v"/>
    </method>
    <method name="Set">
      <arg name="interface" direction="in" type="s"/>
      <arg name="propname" direction="in" type="s"/>
      <arg name="value" direction="in" type="v"/>
    </method>
    <method name="GetAll">
      <arg name="interface" direction="in" type="s"/>
      <arg name="props" direction="out" type="a{sv}"/>
    </method>
  </interface>
  <interface name="org.freedesktop.Notifications">
    <method name="GetServerInformation">
      <arg name="name" type="s" direction="out"/>
      <arg name="vendor" type="s" direction="out"/>
      <arg name="version" type="s" direction="out"/>
    </method>
    <method name="GetCapabilities">
      <arg name="caps" type="as" direction="out"/>
    </method>
    <method name="CloseNotification">
      <arg name="id" type="u" direction="in"/>
    </method>
    <method name="Notify">
      <arg name="app_name" type="s" direction="in"/>
      <arg name="id" type="u" direction="in"/>
      <arg name="icon" type="s" direction="in"/>
      <arg name="summary" type="s" direction="in"/>
      <arg name="body" type="s" direction="in"/>
      <arg name="actions" type="as" direction="in"/>
      <arg name="hints" type="a{sv}" direction="in"/>
      <arg name="timeout" type="i" direction="in"/>
      <arg name="return_id" type="u" direction="out"/>
    </method>
    <signal name="NotificationClosed">
      <arg type="u"/>
      <arg type="u"/>
    </signal>
  </interface>
</node>
"
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384858687888990919293949596979899100101102103
```

# Signal的收发小例子

  我们继续学习D-Bus，参考http://dbus.freedesktop.org/doc/dbus/libdbus-tutorial.html，从底层，即libdbus学习如何发送signal，以及如何监听signal。signal在D-Bus的Daemon中广播，为了提高效率，只发送给向daemon注册要求该singal的对象。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021041811002557.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDQ5ODMxOA==,size_16,color_FFFFFF,t_70#pic_center)
  这个图我画了很久，我希望能够比较形象地说明D-Bus中各种概念的关系。对于程序，第一步需要将应用和D-Bus后台建立连接，也就是和System D-Bus daemon或者Session D-Bus daemon建立连接。一旦建立，daemon会给这条连接分配一个名字，这个名字在system或者session的生命周期是唯一的，即unique connection name，为了方便记忆，可以为这条连接分配一个便于记忆的well-known name。对于信号方式，分配这个名字不是必须的（在method_call中是需要的，我们在下一次学习中谈到），因为在信号的监听中秩序给出Interface的名字和信号名称，在下面的例子中，可以将相关的代码屏蔽掉，不影响运行，但是通常我们都这样处理，尤其在复杂的程序中。在我们的例子中，定义这个BUS name为test.singal.source。当然一个好的名字，为了避免于其他应用重复，应当使用com.mycompany.myfunction之类的名字。 ，而interface的名字，一般前面和connection的BUS name一直。

#### 发送方的小程序（转者已修改）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dbus/dbus.h>
#include <unistd.h>

int send_a_signal(char *sigvalue)
{
	DBusError err;
	DBusConnection *connection;
	DBusMessage *msg;
	DBusMessageIter arg;
	dbus_uint32_t  serial = 0;
	int ret;
	
	//步骤1:建立与D-Bus后台的连接
	
	dbus_error_init(&err);
	
	connection = dbus_bus_get(DBUS_BUS_SESSION, &err );
	if(dbus_error_is_set(&err)){
		fprintf(stderr, "ConnectionErr : %s\n", err.message);
		dbus_error_free(&err);
	}
	if(connection == NULL)
		return -1;
	
	//步骤2:给连接名分配一个well-known的名字作为Bus name，这个步骤不是必须的，可以用if 0来注释着一段代码，我们可以用这个名字来检查，是否已经开启了这个应用的另外的进程。
#if 1
	ret = dbus_bus_request_name(connection, "test.singal.source", DBUS_NAME_FLAG_REPLACE_EXISTING, &err);
	if(dbus_error_is_set(&err)){
		fprintf(stderr,"Name Err :%s\n", err.message);
		dbus_error_free(&err);
	}
	if(ret != DBUS_REQUEST_NAME_REPLY_PRIMARY_OWNER)
		return -1;
#endif
	
	//步骤3:发送一个信号
	//根据图，我们给出这个信号的路径（即可以指向对象），接口，以及信号名，创建一个Message
	if((msg = dbus_message_new_signal("/test/signal/Object", "test.signal.Type", "Test"))== NULL){
		fprintf(stderr, "MessageNULL\n");
		return -1;
	}
	//给这个信号（messge）具体的内容
	dbus_message_iter_init_append(msg, &arg);
	if(!dbus_message_iter_append_basic(&arg, DBUS_TYPE_STRING, &sigvalue)){
		fprintf(stderr, "Out OfMemory!\n");
		return -1;
	}
	
	//步骤4: 将信号从连接中发送
	if(!dbus_connection_send(connection, msg, &serial)){
		fprintf(stderr, "Out of Memory!\n");
		return -1;
	}
	dbus_connection_flush(connection);
	printf("Signal Send\n");
	
	//步骤5: 释放相关的分配的内存。
	dbus_message_unref(msg);
	return 0;
}

int main( int argc, char **argv){
	send_a_signal("Hello,world!");
	return 0;
}
1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162636465666768
```

#### 希望接收该信号的的小程序例子（转者已修改）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dbus/dbus.h>
#include <unistd.h>

void listen_signal()
{
	DBusMessage *msg;
	DBusMessageIter arg;
	DBusConnection *connection;
	DBusError err;
	int ret;
	char * sigvalue;
	
	//步骤1:建立与D-Bus后台的连接
	dbus_error_init(&err);
	connection = dbus_bus_get(DBUS_BUS_SESSION, &err);
	if(dbus_error_is_set(&err)){
		fprintf(stderr, "ConnectionError %s\n", err.message);
		dbus_error_free(&err);
	}
	if(connection == NULL)
	return;
	
	//步骤2:给连接名分配一个可记忆名字test.singal.dest作为Bus name，这个步骤不是必须的,但推荐这样处理
	ret = dbus_bus_request_name(connection, "test.singal.dest", DBUS_NAME_FLAG_REPLACE_EXISTING, &err);
	if(dbus_error_is_set(&err)){
		fprintf(stderr, "Name Error%s\n", err.message);
		dbus_error_free(&err);
	}
	if(ret != DBUS_REQUEST_NAME_REPLY_PRIMARY_OWNER)
	return;
	
	//步骤3:通知D-Bus daemon，希望监听来行接口test.signal.Type的信号
	dbus_bus_add_match(connection, "type='signal', interface='test.signal.Type'", &err);
	//实际需要发送东西给daemon来通知希望监听的内容，所以需要flush
	dbus_connection_flush(connection);
	if(dbus_error_is_set(&err)){
		fprintf(stderr, "Match Error%s\n", err.message);
		dbus_error_free(&err);
	}
	
	//步骤4:在循环中监听，每隔开1秒，就去试图自己的连接中获取这个信号。这里给出的是从连接中获取任何消息的方式，所以获取后去检查一下这个消息是否我们期望的信号，并获取内容。我们也可以通过这个方式来获取method call消息。
	while(1){
		dbus_connection_read_write(connection, 0);
		msg = dbus_connection_pop_message(connection);
		if(msg == NULL){
			sleep(1);
			continue;
		}
	
		if(dbus_message_is_signal(msg, "test.signal.Type", "Test")){
			if(!dbus_message_iter_init(msg, &arg))
				fprintf(stderr, "MessageHas no Param");
			else if(dbus_message_iter_get_arg_type(&arg) != DBUS_TYPE_STRING)
				fprintf(stderr, "Param isnot string");
			else
				dbus_message_iter_get_basic(&arg, &sigvalue);
			printf("Got Singal withvalue : %s\n", sigvalue);
		}
		dbus_message_unref(msg);
	}//End of while
}

int main(int argc, char **argv){
	listen_signal();
	return 0;
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869
```

# Method的收发小例子

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210418110741278.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDQ5ODMxOA==,size_16,color_FFFFFF,t_70#pic_center)

#### 监听Method call消息，并返回Method reply消息（转者已修改）

  Method的监听和signal的监听的处理时一样，但是信号是不需要答复，而Method需要。在下面的例子中，我们将学习如何在消息中加入多个参数（在D-Bus学习（四）中，我们加入了一个参数）的情况。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dbus/dbus.h>
#include <unistd.h>

void reply_to_method_call(DBusMessage *msg, DBusConnection *conn){
	DBusMessage *reply;
	DBusMessageIter arg;
	char *param = NULL;
	dbus_bool_t stat = TRUE;
	dbus_uint32_t level = 2010;
	dbus_uint32_t serial = 0;
	
	//从msg中读取参数，这个在上一次学习中学过
	if(!dbus_message_iter_init(msg, &arg))
		printf("Message has noargs\n");
	else if(dbus_message_iter_get_arg_type(&arg) != DBUS_TYPE_STRING)
		printf("Arg is notstring!\n");
	else
		dbus_message_iter_get_basic(&arg, &param);
	if(param == NULL) return;
	
	
	//创建返回消息reply
	reply = dbus_message_new_method_return(msg);
	//在返回消息中填入两个参数，和信号加入参数的方式是一样的。这次我们将加入两个参数。
	dbus_message_iter_init_append(reply, &arg);
	if(!dbus_message_iter_append_basic(&arg, DBUS_TYPE_BOOLEAN, &stat)){
		printf("Out ofMemory!\n");
		exit(1);
	}
	if(!dbus_message_iter_append_basic(&arg, DBUS_TYPE_UINT32, &level)){
		printf("Out ofMemory!\n");
		exit(1);
	}
	//发送返回消息
	if(!dbus_connection_send(conn, reply, &serial)){
		printf("Out of Memory\n");
		exit(1);
	}
	dbus_connection_flush(conn);
	dbus_message_unref(reply);
}


void listen_dbus()
{
	DBusMessage *msg;
	DBusMessageIter arg;
	DBusConnection *connection;
	DBusError err;
	int ret;
	char *sigvalue;
	
	dbus_error_init(&err);
	//创建于session D-Bus的连接
	connection = dbus_bus_get(DBUS_BUS_SESSION, &err);
	if(dbus_error_is_set(&err)){
		fprintf(stderr, "ConnectionError %s\n", err.message);
		dbus_error_free(&err);
	}
	if(connection == NULL)
		return;
	//设置一个BUS name：test.wei.dest
	ret = dbus_bus_request_name(connection, "test.wei.dest", DBUS_NAME_FLAG_REPLACE_EXISTING, &err);
	if(dbus_error_is_set(&err)){
		fprintf(stderr, "Name Error%s\n", err.message);
		dbus_error_free(&err);
	}
	if(ret != DBUS_REQUEST_NAME_REPLY_PRIMARY_OWNER)
		return;
	
	//要求监听某个singal：来自接口test.signal.Type的信号
	dbus_bus_add_match(connection, "type='signal', interface='test.signal.Type'", &err);
	dbus_connection_flush(connection);
	if(dbus_error_is_set(&err)){
		fprintf(stderr, "Match Error%s\n", err.message);
		dbus_error_free(&err);
	}
	
	while(1){
		dbus_connection_read_write(connection, 0);
		msg = dbus_connection_pop_message(connection);
	
		if(msg == NULL){
			sleep(1);
			continue;
		}
	
		if(dbus_message_is_signal(msg, "test.signal.Type", "Test")){
			if(!dbus_message_iter_init(msg, &arg))
				fprintf(stderr, "Message Has no Param");
			else if(dbus_message_iter_get_arg_type(&arg) != DBUS_TYPE_STRING)
				fprintf(stderr, "Param isnot string");
			else{
                dbus_message_iter_get_basic(&arg, &sigvalue);
                fprintf(stdout, "[method_call]Got Singal withvalue : %s\n", sigvalue);
            }
		}else if(dbus_message_is_method_call(msg, "test.method.Type", "Method")){
			//我们这里面先比较了接口名字和方法名字，实际上应当现比较路径
			if(strcmp(dbus_message_get_path(msg), "/test/method/Object") == 0){
				reply_to_method_call(msg, connection);
                fprintf(stdout, "[method_call]Got method_call, reply to it!\n");
            }
		}
		dbus_message_unref(msg);
	}
}

int main(int argc, char **argv){
	listen_dbus();
	return 0;
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384858687888990919293949596979899100101102103104105106107108109110111112113114
```

#### 发送Method call消息，并等待Method reply消息（转者已修改）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dbus/dbus.h>
#include <unistd.h>

//建立与session D-Bus daemo的连接，并设定连接的名字，相关的代码已经多次使用过了
DBusConnection* connect_dbus()
{
	DBusError err;
	DBusConnection *connection;
	int ret;
	
	//Step 1: connecting session bus
	
	dbus_error_init(&err);
	
	connection = dbus_bus_get(DBUS_BUS_SESSION, &err);
	if(dbus_error_is_set(&err)){
		fprintf(stderr, "ConnectionErr : %s\n", err.message);
		dbus_error_free(&err);
	}
	if(connection == NULL)
		return NULL;
	
	//step 2: 设置BUS name，也即连接的名字。
	ret = dbus_bus_request_name(connection, "test.wei.source", DBUS_NAME_FLAG_REPLACE_EXISTING, &err);
	if(dbus_error_is_set(&err)){
		fprintf(stderr, "Name Err :%s\n", err.message);
		dbus_error_free(&err);
	}
	if(ret != DBUS_REQUEST_NAME_REPLY_PRIMARY_OWNER)
		return NULL;
	
	return connection;
}

void send_a_method_call(DBusConnection *connection,char *param)
{
	DBusError err;
	DBusMessage *msg;
	DBusMessageIter arg;
	DBusPendingCall *pending;
	dbus_bool_t *stat;
	dbus_uint32_t *level;
	
	dbus_error_init(&err);
	
	//针对目的地地址，请参考图，创建一个method call消息。Constructs a new message to invoke a method on a remote object.
	msg = dbus_message_new_method_call("test.wei.dest", "/test/method/Object", "test.method.Type", "Method");
	if(msg == NULL){
		fprintf(stderr, "MessageNULL");
		return;
	}
	
	//为消息添加参数。Appendarguments
	dbus_message_iter_init_append(msg, &arg);
	if(!dbus_message_iter_append_basic(&arg, DBUS_TYPE_STRING, &param)){
		fprintf(stderr, "Out of Memory!");
		exit(1);
	}
	
	//发送消息并获得reply的handle。Queues amessage to send, as withdbus_connection_send() , but also returns aDBusPendingCall used to receive a reply to the message.
	if(!dbus_connection_send_with_reply(connection, msg, &pending, -1)){
		fprintf(stderr, "Out of Memory!");
		exit(1);
	}
	
	if(pending == NULL){
		fprintf(stderr, "Pending CallNULL: connection is disconnected ");
		dbus_message_unref(msg);
		return;
	}
	
	dbus_connection_flush(connection);
	dbus_message_unref(msg);
	
	//waiting a reply，在发送的时候，已经获取了methodreply的handle，类型为DBusPendingCall。
	// block until we recieve a reply， Block until the pendingcall is completed.
	dbus_pending_call_block(pending);
	//get the reply message，Gets thereply, or returns NULL if none has been received yet.
	msg = dbus_pending_call_steal_reply(pending);
	if (msg == NULL) {
		fprintf(stderr, "ReplyNull\n");
		exit(1);
	}
	// free the pendingmessage handle
	dbus_pending_call_unref(pending);
	// read the parameters
	if(!dbus_message_iter_init(msg, &arg))
		fprintf(stderr, "Message hasno arguments!\n");
	else if (dbus_message_iter_get_arg_type(&arg) != DBUS_TYPE_BOOLEAN)
		fprintf(stderr, "Argument isnot boolean!\n");
	else
		dbus_message_iter_get_basic(&arg, &stat);
	
	if (!dbus_message_iter_next(&arg))
		fprintf(stderr, "Message hastoo few arguments!\n");
	else if (dbus_message_iter_get_arg_type(&arg) != DBUS_TYPE_UINT32 )
		fprintf(stderr, "Argument isnot int!\n");
	else
		dbus_message_iter_get_basic(&arg, &level);
	
	printf("Got Reply: %d,%d\n", stat, level);
	dbus_message_unref(msg);
}

int main(int argc, char **argv)
{
	DBusConnection *connection;
	connection = connect_dbus();
	if(connection == NULL)
		return -1;
	
	send_a_method_call(connection,"Hello, D-Bus");
	return 0;
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384858687888990919293949596979899100101102103104105106107108109110111112113114115116117
```



#### 测试结果（转者附，dbus-daemon的使用解析见下一篇介绍）

```powershell
[root@imx6ull]:/work/dbus_test# dbus-daemon --session --print-address --fork --print-pid
unix:abstract=/tmp/dbus-SUr1DUGuJO,guid=f5933f6bc7a0ddb591ca9eba00004845
139
[root@imx6ull]:/work/dbus_test#
[root@imx6ull]:/work/dbus_test# export DBUS_SESSION_BUS_ADDRESS='unix:abstract=/tmp/dbus-SUr1DUGuJO,guid=f5933f6bc7a0ddb591ca9eba00004845'
[root@imx6ull]:/work/dbus_test#
[root@imx6ull]:/work/dbus_test# ./signal_recieve &
[root@imx6ull]:/work/dbus_test# ./signal_send
Signal Send
[root@imx6ull]:/work/dbus_test# Got Singal withvalue : Hello,world!

[root@imx6ull]:/work/dbus_test# ./method_recieve &
[root@imx6ull]:/work/dbus_test# ./method_send
[method_call]Got method_call, reply to it!
Got Reply: 1,2010
[root@imx6ull]:/work/dbus_test# 
[root@imx6ull]:/work/dbus_test# ./signal_send
Signal Send
[root@imx6ull]:/work/dbus_test# Got Singal withvalue : Hello,world!
[method_call]Got Singal withvalue : Hello,world!

[root@imx6ull]:/work/dbus_test#
12345678910111213141516171819202122
```

以上测试中，先是使用signal进行测试，然后又进行了method_call测试，均符合预期。然后后面继续使用signal_send发送一个signal信号，这时由于signal和method_call的接收方都在后台运行，而method_call的接收测试程序中也能接收到signal信号，所以都打印了出来。