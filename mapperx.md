# ObjectMapper

功能介绍：http://10.53.19.12/openbmc/docs/-/blob/master/architecture/object-mapper.md

## 服务运行

服务运行参数：/usr/bin/mapperx --service-namespaces=xyz.openbmc_project org.openbmc --interface-namespaces=xyz.openbmc_project org.freedesktop.DBus.ObjectManager org.openbmc --service-blacklists=

现在服务代码中用到的只有service-namespace

## 代码解析

***先介绍一些通用的接口，之后再分析服务本身***

### 注册服务、接口、方法，异步调用的实现

在mapper源码中有以下代码块

```c++
    boost::asio::io_context io;	//这个是boost库提供的用于异步编程的接口
    std::shared_ptr<sdbusplus::asio::connection> systemBus =
        std::make_shared<sdbusplus::asio::connection>(io);
```

```c++
    sdbusplus::asio::object_server server(systemBus);

    // Construct a signal set registered for process termination.
    boost::asio::signal_set signals(io, SIGINT, SIGTERM);
    signals.async_wait(
        [&io](const boost::system::error_code&, int) { io.stop(); });
```

```c++
    std::shared_ptr<sdbusplus::asio::dbus_interface> iface =
        server.add_interface("/xyz/openbmc_project/object_mapper",
                             "xyz.openbmc_project.ObjectMapper");
```

```c++
    //这里注册方法，参数[方法名，函数]
	iface->register_method(
        "GetAncestors", [&interfaceMap](std::string& reqPath,
                                        std::vector<std::string>& interfaces) {
            return getAncestors(interfaceMap, reqPath, interfaces);
        });
```

```c++
    iface->initialize();//register_method可能会有多个，之前注册的方法统一在这里完成
```

```c++
    systemBus->request_name("xyz.openbmc_project.ObjectMapper");

    io.run();
```

### ```sdbusplus::bus::match::match```

```c++
//原型：对库函数sd_bus_add_match的再封装，这里将期待接收的信号类型和回调函数绑定。
//来自模块sdbusplus。
struct match : private sdbusplus::bus::details::bus_friend
{
    /* Define all of the basic class operations:
     *     Not allowed:
     *         - Default constructor to avoid nullptrs.
     *         - Copy operations due to internal unique_ptr.
     *     Allowed:
     *         - Move operations.
     *         - Destructor.
     */
    match() = delete;
    match(const match&) = delete;
    match& operator=(const match&) = delete;
    match(match&&) = default;
    match& operator=(match&&) = default;
    ~match() = default;
    /** @brief Register a signal match.
     *
     *  @param[in] bus - The bus to register on.
     *  @param[in] match - The match to register.
     *  @param[in] handler - The callback for matches.
     *  @param[in] context - An optional context to pass to the handler.
     */
    match(sdbusplus::bus_t& bus, const char* match,
          sd_bus_message_handler_t handler, void* context = nullptr)
    {
        sd_bus_slot* slot = nullptr;
        sd_bus_add_match(get_busp(bus), &slot, match, handler, context);
        _slot = std::move(slot);
    }
    match(sdbusplus::bus_t& bus, const std::string& _match,
          sd_bus_message_handler_t handler, void* context = nullptr) :
        match(bus, _match.c_str(), handler, context)
    {}
    using callback_t = std::function<void(sdbusplus::message_t&)>;
    /** @brief Register a signal match.
     *
     *  @param[in] bus - The bus to register on.
     *  @param[in] match - The match to register.
     *  @param[in] callback - The callback for matches.
     */
    match(sdbusplus::bus_t& bus, const char* match, callback_t callback) :
        _callback(std::make_unique<callback_t>(std::move(callback)))
    {
        sd_bus_slot* slot = nullptr;
        sd_bus_add_match(get_busp(bus), &slot, match, callCallback,
                         _callback.get());
        _slot = std::move(slot);
    }
    match(sdbusplus::bus_t& bus, const std::string& _match,
          callback_t callback) :
        match(bus, _match.c_str(), callback)
    {}
  private:
    slot_t _slot{};
    std::unique_ptr<callback_t> _callback = nullptr;
    static int callCallback(sd_bus_message* m, void* context,
                            sd_bus_error* /*e*/)
    {
        auto c = static_cast<callback_t*>(context);
        message_t message{m};
        (*c)(message);
        return 0;
    }
};

```

man手册:

意思是定义一种消息匹配规则，并将规则和回调函数绑定。这样，当收到符合规则的消息时，回调函数就会执行。

> int sd_bus_add_match(sd_bus *bus, sd_bus_slot **slot, const char *match, sd_bus_message_handler_t callback, void *userdata);
>
> sd_bus_add_match() installs a match rule for messages received on the specified bus connection object bus. The syntax of the match rule expression passed in match is described in the D-Bus Specification[1]. The specified handler function callback is called for each incoming message matching the specified expression, the userdata parameter is passed as-is to the callback function. The match is installed synchronously when connected to a bus broker, i.e. the call sends a control message requested the match to be added to the broker and waits until the broker confirms the match has been installed successfully.

mapperx服务中调用如下。

```c++
	//将函数nameChangeHandler和消息类型绑定。
    sdbusplus::bus::match::match nameOwnerChanged(
        static_cast<sdbusplus::bus::bus&>(*systemBus),
        sdbusplus::bus::match::rules::nameOwnerChanged(), nameChangeHandler);

```

上述```sdbusplus::bus::match::rules::nameOwnerChanged()```是一个string类型的对象。如下：

```c++
inline auto nameOwnerChanged()
{
    return "type='signal',"
           "sender='org.freedesktop.DBus',"
           "member='NameOwnerChanged',"s;
}
//类似的还有
inline auto interfacesAdded()
{
    return "type='signal',"
           "interface='org.freedesktop.DBus.ObjectManager',"
           "member='InterfacesAdded',"s;
}
inline auto interfacesRemoved()
{
    return "type='signal',"
           "interface='org.freedesktop.DBus.ObjectManager',"
           "member='InterfacesRemoved',"s;
}

```

使用dbus-monitor监控匹配的信号。

```c++
/*前两个信号好像是默认会打印出来的
root@palos:~# dbus-monitor --system type=signal,\
> sender=org.freedesktop.DBus,\
> member=NameOwnerChanged
signal time=138526.792173 sender=org.freedesktop.DBus -> destination=:1.3074 serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameAcquired
   string ":1.3074"
signal time=138526.798030 sender=org.freedesktop.DBus -> destination=:1.3074 serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameLost
   string ":1.3074"
signal time=138531.023094 sender=org.freedesktop.DBus -> destination=(null destination) serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameOwnerChanged
   string ":1.3075"
   string ""
   string ":1.3075"
signal time=138531.032194 sender=org.freedesktop.DBus -> destination=(null destination) serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameOwnerChanged
   string "xyz.openbmc_project.CPUSensor"
   string ""
   string ":1.3075"
signal time=138533.229897 sender=org.freedesktop.DBus -> destination=(null destination) serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameOwnerChanged
   string "xyz.openbmc_project.CPUSensor"
   string ":1.3075"
   string ""
signal time=138533.242716 sender=org.freedesktop.DBus -> destination=(null destination) serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameOwnerChanged
   string ":1.3075"
   string ":1.3075"
   string ""
...
*/
/*
root@palos:~# dbus-monitor --system type=signal,\
> interface=org.freedesktop.DBus.ObjectManager,\
> member=InterfacesAdded
signal time=104187.668150 sender=org.freedesktop.DBus -> destination=:1.3954 serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameAcquired
   string ":1.3954"
signal time=104187.670765 sender=org.freedesktop.DBus -> destination=:1.3954 serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameLost
   string ":1.3954"
signal time=104199.179232 sender=:1.34 -> destination=(null destination) serial=150551 path=/xyz/openbmc_project/network; interface=org.freedesktop.DBus.ObjectManager; member=InterfacesAdded
   object path "/xyz/openbmc_project/network/eth0"
   array [
      dict entry(
         string "org.freedesktop.DBus.Peer"
         array [
         ]
      )
      dict entry(
         string "org.freedesktop.DBus.Introspectable"
         array [
         ]
      )
      dict entry(
         string "org.freedesktop.DBus.Properties"
         array [
         ]
      )
      dict entry(
         string "xyz.openbmc_project.Collection.DeleteAll"
         array [
         ]
      )
      dict entry(
         string "xyz.openbmc_project.Network.Neighbor.CreateStatic"
         array [
         ]
      )
      dict entry(
         string "xyz.openbmc_project.Network.IP.Create"
         array [
         ]
      )
      dict entry(
         string "xyz.openbmc_project.Network.MACAddress"
         array [
            dict entry(
               string "MACAddress"
               variant                   string "92:a2:39:2a:37:45"
            )
         ]
      )
      dict entry(
         string "xyz.openbmc_project.Network.EthernetInterface"
         array [
            dict entry(
               string "InterfaceName"
               variant                   string "eth0"
            )
            dict entry(
               string "Speed"
               variant                   uint32 0
            )
            dict entry(
               string "AutoNeg"
               variant                   boolean false
            )
            dict entry(
               string "MTU"
               variant                   uint32 1500
            )
            dict entry(
               string "DomainName"
               variant                   array [
                  ]
            )
            dict entry(
               string "DHCPEnabled"
               variant                   string "xyz.openbmc_project.Network.EthernetInterface.DHCPConf.both"
            )
            dict entry(
               string "Nameservers"
               variant                   array [
                  ]
            )
            dict entry(
               string "StaticNameServers"
               variant                   array [
                  ]
            )
            dict entry(
               string "NTPServers"
               variant                   array [
                  ]
            )
            dict entry(
               string "StaticNTPServers"
               variant                   array [
                  ]
            )
            dict entry(
               string "LinkLocalAutoConf"
               variant                   string "xyz.openbmc_project.Network.EthernetInterface.LinkLocalConf.fallback"
            )
            dict entry(
               string "IPv6AcceptRA"
               variant                   boolean true
            )
            dict entry(
               string "NICEnabled"
               variant                   boolean true
            )
            dict entry(
               string "LinkUp"
               variant                   boolean true
            )
            dict entry(
               string "DefaultGateway"
               variant                   string "192.100.1.200"
            )
            dict entry(
               string "DefaultGateway6"
               variant                   string "fe80::20c:29ff:fe53:a412"
            )
         ]
      )
   ]
...
*/
/*
root@palos:~# dbus-monitor --system type=signal,\
> interface=org.freedesktop.DBus.ObjectManager,\
> member=InterfacesRemoved
signal time=105514.514905 sender=org.freedesktop.DBus -> destination=:1.4018 serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameAcquired
   string ":1.4018"
signal time=105514.517281 sender=org.freedesktop.DBus -> destination=:1.4018 serial=4294967295 path=/org/freedesktop/DBus; interface=org.freedesktop.DBus; member=NameLost
   string ":1.4018"
signal time=105534.041356 sender=:1.34 -> destination=(null destination) serial=152467 path=/xyz/openbmc_project/network; interface=org.freedesktop.DBus.ObjectManager; member=InterfacesRemoved
   object path "/xyz/openbmc_project/network/usb0"
   array [
      string "org.freedesktop.DBus.Peer"
      string "org.freedesktop.DBus.Introspectable"
      string "org.freedesktop.DBus.Properties"
      string "xyz.openbmc_project.Collection.DeleteAll"
      string "xyz.openbmc_project.Network.Neighbor.CreateStatic"
      string "xyz.openbmc_project.Network.IP.Create"
      string "xyz.openbmc_project.Network.MACAddress"
      string "xyz.openbmc_project.Network.EthernetInterface"
   ]
signal time=105534.069441 sender=:1.34 -> destination=(null destination) serial=152468 path=/xyz/openbmc_project/network; interface=org.freedesktop.DBus.ObjectManager; member=InterfacesRemoved
   object path "/xyz/openbmc_project/network/eth1"
   array [
      string "org.freedesktop.DBus.Peer"
      string "org.freedesktop.DBus.Introspectable"
      string "org.freedesktop.DBus.Properties"
      string "xyz.openbmc_project.Collection.DeleteAll"
      string "xyz.openbmc_project.Network.Neighbor.CreateStatic"
      string "xyz.openbmc_project.Network.IP.Create"
      string "xyz.openbmc_project.Network.MACAddress"
      string "xyz.openbmc_project.Network.EthernetInterface"
   ]
*/
```

***接下来分析服务代码***

在main函数中有以下代码块

```c++
	//post方法向io提交一个事务，在服务运行的时候会被调用，其参数是一个函数对象。
	/*interfaceMap是一个以引用传递的变量，用来存放子树的，doListNames就是为了给初始化
	这个变量。
	*/
	io.post([&]() {	
        doListNames(io, interfaceMap, systemBus.get(), nameOwners,
                    associationMaps, server);
    });
```

###分析doListNames方法

```c++
void doListNames(
    boost::asio::io_context& io, InterfaceMapType& interfaceMap,
    sdbusplus::asio::connection* systemBus,
    boost::container::flat_map<std::string, std::string>& nameOwners,
    AssociationMaps& assocMaps, sdbusplus::asio::object_server& objectServer)
{
    //通过调用dbus上的服务，获取dbus上所有的服务名称，用processNames去接收
    systemBus->async_method_call(
        [&io, &interfaceMap, &nameOwners, &objectServer, systemBus,
         &assocMaps](const boost::system::error_code ec,
                     std::vector<std::string> processNames) {
            if (ec)
            {
                std::cerr << "Error getting names: " << ec << "\n";
                std::exit(EXIT_FAILURE);
                return;
            }
            // Try to make startup consistent
            std::sort(processNames.begin(), processNames.end());
#ifdef MAPPER_ENABLE_DEBUG
            std::shared_ptr<std::chrono::time_point<std::chrono::steady_clock>>
                globalStartTime = std::make_shared<
                    std::chrono::time_point<std::chrono::steady_clock>>(
                    std::chrono::steady_clock::now());
#endif
            for (const std::string& processName : processNames)
            {
                //serviceAllowList是服务启动传入的参数，用来过滤符合要求的服务名称
                if (needToIntrospect(processName, serviceAllowList))
                {
                    /*符合要求的函数进行以下调用，startNewIntrospect函数内部去调用
                    systemBus->async_method_call时会立即返回，不会阻塞，然后继续执行
                    updateOwners。
                    */
                    startNewIntrospect(systemBus, io, interfaceMap, processName,
                                       assocMaps,
#ifdef MAPPER_ENABLE_DEBUG
                                       globalStartTime,
#endif
                                       objectServer);
                    updateOwners(systemBus, nameOwners, processName);
                }
            }
        },
        "org.freedesktop.DBus", "/org/freedesktop/DBus", "org.freedesktop.DBus",
        "ListNames");
}

```

获取dbus上的所有服务名称

```c++
/*
root@palos:~# busctl call org.freedesktop.DBus /org/freedesktop/DBus \
> org.freedesktop.DBus ListNames
as 102 "org.freedesktop.DBus" ":1.0" ":1.1" ":1.2" ":1.3" ":1.4" ":1.5" ":1.6" ":1.7" ":1.8" ":1.9" ":1.10" ":1.11" ":1.12" ":1.14" ":1.15" ":1.16" ":1.17" ":1.18" ":1.19" ":1.20" ":1.21" ":1.22" ":1.23" ":1.24" ":1.25" ":1.26" ":1.27" ":1.28" ":1.29" ":1.30" ":1.31" ":1.32" ":1.33" ":1.34" ":1.35" ":1.36" ":1.37" ":1.38" ":1.39" ":1.41" ":1.43" ":1.44" ":1.46" ":1.47" ":1.49" ":1.50" ":1.51" ":1.52" ":1.53" ":1.54" ":1.55" ":1.56" ":1.59" ":1.61" ":1.70" ":1.114" ":1.115" ":1.123" "org.freedesktop.Avahi" "org.freedesktop.network1" "org.freedesktop.resolve1" "org.freedesktop.systemd1" "org.freedesktop.timesync1" "xyz.openbmc_project.ADCSensor" "xyz.openbmc_project.Certs.Manager.Authority.Ldap" "xyz.openbmc_project.Certs.Manager.Client.Ldap" "xyz.openbmc_project.Certs.Manager.Server.Https" "xyz.openbmc_project.Dump.Manager" "xyz.openbmc_project.EntityManager" "xyz.openbmc_project.ExitAirTempSensor" "xyz.openbmc_project.ExternalSensor" "xyz.openbmc_project.FanSensor" "xyz.openbmc_project.FruDevice" "xyz.openbmc_project.HealthMon" "xyz.openbmc_project.Hwmon.external" "xyz.openbmc_project.HwmonTempSensor" "xyz.openbmc_project.IntrusionSensor" "xyz.openbmc_project.IpmbSensor" "xyz.openbmc_project.LED.Controller.bmc_alive" "xyz.openbmc_project.LED.Controller.health_green" "xyz.openbmc_project.LED.Controller.health_red" "xyz.openbmc_project.LED.Controller.uid" "xyz.openbmc_project.Ldap.Config" "xyz.openbmc_project.Logging" "xyz.openbmc_project.Logging.IPMI" "xyz.openbmc_project.MCUTempSensor" "xyz.openbmc_project.Network" "xyz.openbmc_project.ObjectMapper" "xyz.openbmc_project.PSUSensor" "xyz.openbmc_project.Settings" "xyz.openbmc_project.Software.BMC.Updater" "xyz.openbmc_project.Software.Download" "xyz.openbmc_project.Software.Version" "xyz.openbmc_project.State.BMC" "xyz.openbmc_project.State.Boot.PostCode0" "xyz.openbmc_project.State.Boot.Raw" "xyz.openbmc_project.State.FanCtrl" "xyz.openbmc_project.Syslog.Config" "xyz.openbmc_project.Telemetry" "xyz.openbmc_project.User.Manager" "xyz.openbmc_project.VirtualSensor"

返回的是系统bus上的服务，返回类型as，字符串数组，由processNames接收。
*/
```

### 分析doIntrospect方法

```c++
void startNewIntrospect(
    sdbusplus::asio::connection* systemBus, boost::asio::io_context& io,
    InterfaceMapType& interfaceMap, const std::string& processName,
    AssociationMaps& assocMaps,
#ifdef MAPPER_ENABLE_DEBUG
    std::shared_ptr<std::chrono::time_point<std::chrono::steady_clock>>
        globalStartTime,
#endif
    sdbusplus::asio::object_server& objectServer)
{
    if (needToIntrospect(processName, serviceAllowList))
    {
        std::shared_ptr<InProgressIntrospect> transaction =
            std::make_shared<InProgressIntrospect>(systemBus, io, processName,
                                                   assocMaps
#ifdef MAPPER_ENABLE_DEBUG
                                                   ,
                                                   globalStartTime
#endif
            );
        //doInrospect才是最终需要解读的函数
        doIntrospect(systemBus, transaction, interfaceMap, objectServer, "/");
    }

}
```

```c++
void doIntrospect(sdbusplus::asio::connection* systemBus,
                  const std::shared_ptr<InProgressIntrospect>& transaction,
                  InterfaceMapType& interfaceMap,
                  sdbusplus::asio::object_server& objectServer,
                  const std::string& path, int timeoutRetries = 0)
{
    //会去调用transaction->processName服务的方法，然后获取所有路径上的接口。
    constexpr int maxTimeoutRetries = 3;
    systemBus->async_method_call(
        [&interfaceMap, &objectServer, transaction, path, systemBus,
         timeoutRetries](const boost::system::error_code ec,
                         const std::string& introspectXml) {
            if (ec)
            {
                if (ec.value() == boost::system::errc::timed_out &&
                    timeoutRetries < maxTimeoutRetries)
                {
                    doIntrospect(systemBus, transaction, interfaceMap,
                                 objectServer, path, timeoutRetries + 1);
                    return;
                }
                std::cerr << "Introspect call failed with error: " << ec << ", "
                          << ec.message()
                          << " on process: " << transaction->processName
                          << " path: " << path << "\n";
                return;
            }
            tinyxml2::XMLDocument doc;
            tinyxml2::XMLError e = doc.Parse(introspectXml.c_str());
            if (e != tinyxml2::XMLError::XML_SUCCESS)
            {
                std::cerr << "XML parsing failed\n";
                return;
            }
            tinyxml2::XMLNode* pRoot = doc.FirstChildElement("node");
            if (pRoot == nullptr)
            {
                std::cerr << "XML document did not contain any data\n";
                return;
            }
            auto& thisPathMap = interfaceMap[path];
            tinyxml2::XMLElement* pElement =
                pRoot->FirstChildElement("interface");
            while (pElement != nullptr)
            {
                const char* ifaceName = pElement->Attribute("name");
                if (ifaceName == nullptr)
                {
                    continue;
                }
                thisPathMap[transaction->processName].emplace(ifaceName);
                if (std::strcmp(ifaceName, assocDefsInterface) == 0)
                {
                    //下面这个方法也很重要
                    doAssociations(systemBus, interfaceMap, objectServer,
                                   transaction->processName, path);
                }
                pElement = pElement->NextSiblingElement("interface");
            }
            // Check if this new path has a pending association that can
            // now be completed.
            checkIfPendingAssociation(path, interfaceMap,
                                      transaction->assocMaps, objectServer);
            pElement = pRoot->FirstChildElement("node");
            while (pElement != nullptr)
            {
                const char* childPath = pElement->Attribute("name");
                if (childPath != nullptr)
                {
                    std::string parentPath(path);
                    if (parentPath == "/")
                    {
                        parentPath.clear();
                    }
                    //递归函数，更新path后继续调用下去。
                    doIntrospect(systemBus, transaction, interfaceMap,
                                 objectServer, parentPath + "/" + childPath);
                }
                pElement = pElement->NextSiblingElement("node");
            }
        },
        transaction->processName, path, "org.freedesktop.DBus.Introspectable",
        "Introspect");
}

/*
看一下具体的调用,本身返回的是一个xml格式的字符串。
root@palos:~# dbus-send --system --print-reply --type=method_call\
> --dest=xyz.openbmc_project.FruDevice / \
> org.freedesktop.DBus.Introspectable.Introspect
method return time=125384.901297 sender=:1.13 -> destination=:1.975 serial=700 reply_serial=2
   string "<!DOCTYPE node PUBLIC "-//freedesktop//DTD D-BUS Object Introspection 1.0//EN"
"http://www.freedesktop.org/standards/dbus/1.0/introspect.dtd">
<node>
 <interface name="org.freedesktop.DBus.Peer">
  <method name="Ping"/>
  <method name="GetMachineId">
   <arg type="s" name="machine_uuid" direction="out"/>
  </method>
 </interface>
 <interface name="org.freedesktop.DBus.Introspectable">
  <method name="Introspect">
   <arg name="xml_data" type="s" direction="out"/>
  </method>
 </interface>
 <interface name="org.freedesktop.DBus.Properties">
  <method name="Get">
   <arg name="interface_name" direction="in" type="s"/>
   <arg name="property_name" direction="in" type="s"/>
   <arg name="value" direction="out" type="v"/>
  </method>
  <method name="GetAll">
   <arg name="interface_name" direction="in" type="s"/>
   <arg name="props" direction="out" type="a{sv}"/>
  </method>
  <method name="Set">
   <arg name="interface_name" direction="in" type="s"/>
   <arg name="property_name" direction="in" type="s"/>
   <arg name="value" direction="in" type="v"/>
  </method>
  <signal name="PropertiesChanged">
   <arg type="s" name="interface_name"/>
   <arg type="a{sv}" name="changed_properties"/>
   <arg type="as" name="invalidated_properties"/>
  </signal>
 </interface>
 <interface name="org.freedesktop.DBus.ObjectManager">
  <method name="GetManagedObjects">
   <arg type="a{oa{sa{sv}}}" name="object_paths_interfaces_and_properties" direction="out"/>
  </method>
  <signal name="InterfacesAdded">
   <arg type="o" name="object_path"/>
   <arg type="a{sa{sv}}" name="interfaces_and_properties"/>
  </signal>
  <signal name="InterfacesRemoved">
   <arg type="o" name="object_path"/>
   <arg type="as" name="interfaces"/>
  </signal>
 </interface>
 <node name="xyz"/>
</node>
"
*/
/*
获取node name的值，后会继续调用：
    dbus-send --system --print-reply --type=method_call --dest=xyz.openbmc_project.FruDevice /xyz org.freedesktop.DBus.Introspectable.Introspect
    dbus-send --system --print-reply --type=method_call --dest=xyz.openbmc_project.FruDevice /xyz/openbmc_project org.freedesktop.DBus.Introspectable.Introspect
    dbus-send --system --print-reply --type=method_call --dest=xyz.openbmc_project.FruDevice /xyz/openbmc_project/FruDevice org.freedesktop.DBus.Introspectable.Introspect
    dbus-send --system --print-reply --type=method_call --dest=xyz.openbmc_project.FruDevice /xyz/openbmc_project/FruDevice/G220A org.freedesktop.DBus.Introspectable.Introspect
    dbus-send --system --print-reply --type=method_call --dest=xyz.openbmc_project.FruDevice /xyz/openbmc_project/FruDevice/13_76 org.freedesktop.DBus.Introspectable.Introspect
    dbus-send --system --print-reply --type=method_call --dest=xyz.openbmc_project.FruDevice /xyz/openbmc_project/FruDevice/12_76 org.freedesktop.DBus.Introspectable.Introspect
    dbus-send --system --print-reply --type=method_call --dest=xyz.openbmc_project.FruDevice /xyz/openbmc_project/FruDevice/4_87 org.freedesktop.DBus.Introspectable.Introspect
    dbus-send --system --print-reply --type=method_call --dest=xyz.openbmc_project.FruDevice /xyz/openbmc_project/FruDevice/3_76 org.freedesktop.DBus.Introspectable.Introspect
*/
/*
interfaceMap的元素的意义：
存储系统总线上所有和openbmc相关的service，并且通过解析xml的形式获取其路径下的所有接口。
其值的形式为map{string(path):map{strign(service):set[interface1,..,interfacen])}}
在一个路径（path）下，某个服务（service）包含的所有接口（interface）。
*/
```

### 分析doAssociations方法

```c++
void doAssociations(sdbusplus::asio::connection* systemBus,
                    InterfaceMapType& interfaceMap,
                    sdbusplus::asio::object_server& objectServer,
                    const std::string& processName, const std::string& path,
                    int timeoutRetries = 0)
{
    constexpr int maxTimeoutRetries = 3;
    systemBus->async_method_call(
        [&objectServer, path, processName, &interfaceMap, systemBus,
         timeoutRetries](
            const boost::system::error_code ec,
            const std::variant<std::vector<Association>>& variantAssociations) {
            if (ec)
            {
                if (ec.value() == boost::system::errc::timed_out &&
                    timeoutRetries < maxTimeoutRetries)
                {
                    doAssociations(systemBus, interfaceMap, objectServer,
                                   processName, path, timeoutRetries + 1);
                    return;
                }
                std::cerr << "Error getting associations from " << path << "\n";
            }
            std::vector<Association> associations =
                std::get<std::vector<Association>>(variantAssociations);
            //下面的方法是主要分析的
            associationChanged(objectServer, associations, path, processName,
                               interfaceMap, associationMaps);
        },
        processName, path, "org.freedesktop.DBus.Properties", "Get",
        assocDefsInterface, assocDefsProperty);

}
//assocDefsInterface："xyz.openbmc_project.Association.Definitions"
//assocDefsProperty: "Associations"
/*
root@palos:~# busctl call xyz.openbmc_project.HealthMon /xyz/openbmc_project/sensors/utilization/CPU org.freedesktop.DBus.Properties Get ss xyz.openbmc_project.Association.Definitions Associations
v a(sss) 0

root@palos:~# busctl call xyz.openbmc_project.VirtualSensor /xyz/openbmc_project/sensors/power/P1_DIMM_VR_Pwr org.freedesktop.DBus.Properties Get ss xyz.openbmc_project.Association.Definitions Associations
v a(sss) 1 "chassis" "all_sensors" "/xyz/openbmc_project/inventory/system/board/Palos"

root@palos:~# busctl call xyz.openbmc_project.Software.BMC.Updater /xyz/openbmc_project/software org.freedesktop.DBus.Properties Get ss xyz.openbmc_project.Association.Definitions Associations
v a(sss) 3 "functional" "software_version" "/xyz/openbmc_project/software/2fc65b6c" "active" "software_version" "/xyz/openbmc_project/software/2fc65b6c" "updateable" "software_version" "/xyz/openbmc_project/software/2fc65b6c"

root@palos:~# busctl call xyz.openbmc_project.Software.BMC.Updater /xyz/openbmc_project/software/2fc65b6c org.freedesktop.DBus.Properties Get ss xyz.openbmc_project.Association.Definitions Associations
v a(sss) 1 "inventory" "activation" ""
*/
```

```c++
void associationChanged(sdbusplus::asio::object_server& objectServer,
                        const std::vector<Association>& associations,
                        const std::string& path, const std::string& owner,
                        const InterfaceMapType& interfaceMap,
                        AssociationMaps& assocMaps)
{
    AssociationPaths objects;
    for (const Association& association : associations)
    {
        std::string forward;
        std::string reverse;
        std::string objectPath;
        std::tie(forward, reverse, objectPath) = association;
        if (objectPath.empty())
        {
            std::cerr << "Found invalid association on path " << path << "\n";
            continue;
        }
        // Can't create this association if the endpoint isn't on D-Bus.
        if (interfaceMap.find(objectPath) == interfaceMap.end())
        {
            addPendingAssociation(objectPath, reverse, path, forward, owner,
                                  assocMaps);
            continue;
        }
        if (!forward.empty())
        {
            //创建关联
            objects[path + "/" + forward].emplace(objectPath);
        }
        if (!reverse.empty())
        {
            //创建关联
            objects[objectPath + "/" + reverse].emplace(path);
        }
    }
    for (const auto& object : objects)
    {
        //添加关联
        addEndpointsToAssocIfaces(objectServer, object.first, object.second,
                                  assocMaps);
    }
    // Check for endpoints being removed instead of added
    checkAssociationEndpointRemoves(path, owner, objects, objectServer,
                                    assocMaps);
    if (!objects.empty())
    {
        // Update associationOwners with the latest info
        auto a = assocMaps.owners.find(path);
        if (a != assocMaps.owners.end())
        {
            auto o = a->second.find(owner);
            if (o != a->second.end())
            {
                o->second = std::move(objects);
            }
            else
            {
                a->second.emplace(owner, std::move(objects));
            }
        }
        else
        {
            boost::container::flat_map<std::string, AssociationPaths> owners;
            owners.emplace(owner, std::move(objects));
            assocMaps.owners.emplace(path, owners);
        }
    }
}
```

```c++
void addEndpointsToAssocIfaces(
    sdbusplus::asio::object_server& objectServer, const std::string& assocPath,
    const boost::container::flat_set<std::string>& endpointPaths,
    AssociationMaps& assocMaps)
{
    auto& iface = assocMaps.ifaces[assocPath];
    auto& i = std::get<ifacePos>(iface);
    auto& endpoints = std::get<endpointsPos>(iface);
    // Only add new endpoints
    for (const auto& e : endpointPaths)
    {
        if (std::find(endpoints.begin(), endpoints.end(), e) == endpoints.end())
        {
            endpoints.push_back(e);
        }
    }
    // If the interface already exists, only need to update
    // the property value, otherwise create it
    if (i)
    {
        i->set_property("endpoints", endpoints);
    }
    else
    {
        //接口不存在的时候，创建接口
        i = objectServer.add_interface(assocPath, xyzAssociationInterface);
        i->register_property("endpoints", endpoints);
        i->initialize();
    }
}
//xyzAssociationInterface: "xyz.openbmc_project.Association";
/*
root@palos:~# busctl introspect xyz.openbmc_project.ObjectMapper /xyz/openbmc_project/software/2fc65b6c/software_version
NAME                                TYPE      SIGNATURE RESULT/VALUE                      FLAGS
org.freedesktop.DBus.Introspectable interface -         -                                 -
.Introspect                         method    -         s                                 -
org.freedesktop.DBus.Peer           interface -         -                                 -
.GetMachineId                       method    -         s                                 -
.Ping                               method    -         -                                 -
org.freedesktop.DBus.Properties     interface -         -                                 -
.Get                                method    ss        v                                 -
.GetAll                             method    s         a{sv}                             -
.Set                                method    ssv       -                                 -
.PropertiesChanged                  signal    sa{sv}as  -                                 -
xyz.openbmc_project.Association     interface -         -                                 -
.endpoints                          property  as        1 "/xyz/openbmc_project/software" emits-change
*/
```

***下面看一下和信号方法相关的代码***

和信号绑定的方法在总线上有匹配的信号时才会被调用。

前面提到的nameChangeHandler这个函数。在收到符合匹配规则的信号后会被调用。

```c++
	//一个函数对象nameChangeHandler，使用message接收信号携带的消息
	std::function<void(sdbusplus::message::message & message)>
        nameChangeHandler = [&interfaceMap, &io, &nameOwners, &server,
                             systemBus](sdbusplus::message::message& message) {
            std::string name;     // well-known
            std::string oldOwner; // unique-name
            std::string newOwner; // unique-name
            message.read(name, oldOwner, newOwner);
            if (!oldOwner.empty())
            {
                processNameChangeDelete(nameOwners, name, oldOwner,
                                        interfaceMap, associationMaps, server);
            }
            if (!newOwner.empty())
            {
#ifdef MAPPER_ENABLE_DEBUG
                auto transaction = std::make_shared<
                    std::chrono::time_point<std::chrono::steady_clock>>(
                    std::chrono::steady_clock::now());
#endif
                // New daemon added
                if (needToIntrospect(name, serviceAllowList))
                {
                    nameOwners[newOwner] = name;
                    //前面已经介绍过了，所有，在总线上的服务可以通过信号的形式主动更新endpoints
                    startNewIntrospect(systemBus.get(), io, interfaceMap, name,
                                       associationMaps,
#ifdef MAPPER_ENABLE_DEBUG
                                       transaction,
#endif
                                       server);
                }
            }
        };
//代码中还有通过其他信号调用的代码，具体形式也是类似的，即通过几种类型的信号更新endpoints
```

mapperx作为服务需要向其他服务提供method，前面提到过，通过register_method去添加，这里介绍一下GetSubTree方法

```c++
	/*以下调用示例中的参数string:"/" int32:0
	array:string:"xyz.openbmc_project.Sensor.Threshold.Warning"分别对应repPath,depth,interfaces
	*/
	iface->register_method(
        "GetSubTree", [&interfaceMap](std::string& reqPath, int32_t depth,
                                      std::vector<std::string>& interfaces) {
            return getSubTree(interfaceMap, reqPath, depth, interfaces);
        });
```

```c++
//interfaceMap前面介绍过它的数据格式，这里就是根据传入的参数丛中筛选出符合的元素并返回。
//其余的几个接口也是同样的道理。
std::vector<InterfaceMapType::value_type>
    getSubTree(const InterfaceMapType& interfaceMap, std::string reqPath,
               int32_t depth, std::vector<std::string>& interfaces)
{
    if (depth <= 0)
    {
        depth = std::numeric_limits<int32_t>::max();
    }
    // Interfaces need to be sorted for intersect to function
    std::sort(interfaces.begin(), interfaces.end());
    std::vector<InterfaceMapType::value_type> ret;

    if (boost::ends_with(reqPath, "/"))
    {
        reqPath.pop_back();
    }
    if (!reqPath.empty() && interfaceMap.find(reqPath) == interfaceMap.end())
    {
        throw sdbusplus::xyz::openbmc_project::Common::Error::
            ResourceNotFound();
    }

    for (const auto& objectPath : interfaceMap)
    {
        const auto& thisPath = objectPath.first;

        if (thisPath == reqPath)
        {
            continue;
        }

        if (boost::starts_with(thisPath, reqPath))
        {
            // count the number of slashes past the search term
            int32_t thisDepth = std::count(thisPath.begin() + reqPath.size(),
                                           thisPath.end(), '/');
            if (thisDepth <= depth)
            {
                for (const auto& interfaceMap : objectPath.second)
                {
                    if (intersect(interfaces.begin(), interfaces.end(),
                                  interfaceMap.second.begin(),
                                  interfaceMap.second.end()) ||
                        interfaces.empty())
                    {
                        addObjectMapResult(ret, thisPath, interfaceMap);
                    }
                }
            }
        }
    }

    return ret;
}
/*以下是一个调用示例
dbus-send --system --print-reply \
--dest=xyz.openbmc_project.ObjectMapper \
/xyz/openbmc_project/object_mapper \
xyz.openbmc_project.ObjectMapper.GetSubTree \
string:"/" int32:0 array:string:"xyz.openbmc_project.Sensor.Threshold.Warning"
   array [
      dict entry(
         string "/xyz/openbmc_project/sensors/current/ps0_output_current"
         array [
            dict entry(
               string "xyz.openbmc_project.Hwmon-1040041051.Hwmon1"
               array [
                  string "xyz.openbmc_project.Sensor.Threshold.Critical"
                  string "xyz.openbmc_project.Sensor.Threshold.Warning"
                  string "xyz.openbmc_project.Sensor.Value"
               ]
            )
         ]
      )
      dict entry(
         string "/xyz/openbmc_project/sensors/current/ps1_output_current"
         array [
            dict entry(
               string "xyz.openbmc_project.Hwmon-1025936882.Hwmon1"
               array [
                  string "xyz.openbmc_project.Sensor.Threshold.Critical"
                  string "xyz.openbmc_project.Sensor.Threshold.Warning"
                  string "xyz.openbmc_project.Sensor.Value"
               ]
            )
         ]
      )
...

*/
```

