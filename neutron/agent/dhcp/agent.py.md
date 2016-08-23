## **agent.py**

---------------------------------

一般来说，neutron中的RPC API分为两个部分，client端和server端，DHCP agent包括一个client API，`neutron.agent.dhcp.agent.DhcpPluginAPI`。DHCP agent利用这个类回调在neutron server端的远程方法，这个server端定义在`neutron.api.rpc.handlers.dhcp_rpc.DhcpRpcCallback`，通常由 neutron plugin来决定是否暴露这个DhcpRpcCallback接口。


### **DhcpAgent类**

该类主要继承自`manager.Manager`，该类中的方法主要实现了RPC接口中的server端，neutron server（RESTful API）主要利用`neutron.api.rpc.agentnotifiers.dhcp_rpc_agent_api.DhcpAgentNotifyApi`来作为client端执行这里的方法。

```
# RPC消息通信方式，一般是RabbitMQ实现
target = oslo_messaging.Target(version='1.0')

```

初始化的方法主要读取配置文件中的driver参数。

```
    def __init__(self, host=None, conf=None):
        super(DhcpAgent, self).__init__(host=host)
        self.needs_resync_reasons = collections.defaultdict(list)
		# 读取配置信息conf
        self.conf = conf or cfg.CONF
        self.cache = NetworkCache()
        self.dhcp_driver_cls = importutils.import_class(self.conf.dhcp_driver)
        ctx = context.get_admin_context_without_session()
		# 根据这个创建的DhcpPluginApi类，应该是作为client端向plugin发送RPC消息，topic类型为topics.PLUGIN
        self.plugin_rpc = DhcpPluginApi(topics.PLUGIN,
                                        ctx, self.conf.use_namespaces,
                                        self.conf.host)
        # create dhcp dir to store dhcp info
		# DHCP路径 state_path/dhcp
        dhcp_dir = os.path.dirname("/%s/dhcp/" % self.conf.state_path)
        utils.ensure_dir(dhcp_dir)
		# 检查版本信息
        self.dhcp_version = self.dhcp_driver_cls.check_version()
        self._populate_networks_cache()
        self._process_monitor = external_process.ProcessMonitor(
            config=self.conf,
            resource_type='dhcp')

```




```
    def init_host(self):
        self.sync_state()

```



`after_start()`方法调用run()方法，主要执行两个动作，一是将neutron中状态同步到本地，另外就是创建一个新的协程来周期性的同步状态。


```
    def after_start(self):
        self.run()
        LOG.info(_LI("DHCP agent started"))

    def run(self):
        """Activate the DHCP agent."""
		# 同步neutron的状态
        self.sync_state()
		# 周期性的同步状态
        self.periodic_resync()

```

作为client端，调用`sync_state()`会发出rpc消息给server端的plugin，通过DhcpRpcCallback获取最新的网络状态，然后更新本地信息，调用dnsmasq进程使之生效。该方法在启动后运行一次。 `periodic_resync()`方法则孵化一个协程来运行`_periodic_resync_helper()`方法，该函数是一个无限循环，它周期性的调用`sync_state()`。

#### `sync_state()`

```
    def sync_state(self, networks=None):
        """Sync the local DHCP state with Neutron. If no networks are passed,
        or 'None' is one of the networks, sync all of the networks.
        """
        only_nets = set([] if (not networks or None in networks) else networks)
        LOG.info(_LI('Synchronizing state'))
		# 创建一个协程池
        pool = eventlet.GreenPool(self.conf.num_sync_threads)
		# 得到known_network_ids，active_network_ids计算出deleted_id，来获取最新的网络状态
        known_network_ids = set(self.cache.get_network_ids())

        try:
            active_networks = self.plugin_rpc.get_active_networks_info()
            active_network_ids = set(network.id for network in active_networks)
            for deleted_id in known_network_ids - active_network_ids:
                try:
                    self.disable_dhcp_helper(deleted_id)
                except Exception as e:
                    self.schedule_resync(e, deleted_id)
                    LOG.exception(_LE('Unable to sync network state on '
                                      'deleted network %s'), deleted_id)

            for network in active_networks:
                if (not only_nets or  # specifically resync all
                        network.id not in known_network_ids or  # missing net
                        network.id in only_nets):  # specific network to sync
                    pool.spawn(self.safe_configure_dhcp_for_network, network)
            pool.waitall()
            LOG.info(_LI('Synchronizing state complete'))

        except Exception as e:
            if only_nets:
                for network_id in only_nets:
                    self.schedule_resync(e, network_id)
            else:
                self.schedule_resync(e)
            LOG.exception(_LE('Unable to sync network state.'))

```


#### `_periodic_resync_helper()`



```
    @utils.exception_logger()
    def _periodic_resync_helper(self):
        """Resync the dhcp state at the configured interval."""
		# 无限循环
        while True:
		# 协程的休眠时间为同步间隔
            eventlet.sleep(self.conf.resync_interval)
            if self.needs_resync_reasons:
                # be careful to avoid a race with additions to list
                # from other threads
                reasons = self.needs_resync_reasons
                self.needs_resync_reasons = collections.defaultdict(list)
                for net, r in reasons.items():
                    if not net:
                        net = "*"
                    LOG.debug("resync (%(network)s): %(reason)s",
                              {"reason": r, "network": net})
		# 周期间隔调用sync_state()
                self.sync_state(reasons.keys())

```



```
	# 生成同步的协程
    def periodic_resync(self):
        """Spawn a thread to periodically resync the dhcp state."""
        eventlet.spawn(self._periodic_resync_helper)

```


### **DhcpPluginApi类**

主要实现RPC通信的client端，向server端的plugin发送RPC消息，plugin通过`neutron.api.rpc.handlers.dhcp_rpc.DhcpRpcCallback`来实现接口

```
    def __init__(self, topic, context, use_namespaces, host):
        self.context = context
        self.host = host
        self.use_namespaces = use_namespaces
		# RPC消息实例的实现，获得client端的实例
        target = oslo_messaging.Target(
                topic=topic,
                namespace=constants.RPC_NAMESPACE_DHCP_PLUGIN,
                version='1.0')
        self.client = n_rpc.get_client(target)

```

主要包含了几个方法

```
    def get_active_networks_info(self):
        """Make a remote process call to retrieve all network info."""
        cctxt = self.client.prepare(version='1.1')
        networks = cctxt.call(self.context, 'get_active_networks_info',
                              host=self.host)
        return [dhcp.NetModel(self.use_namespaces, n) for n in networks]

    def get_network_info(self, network_id):
        """Make a remote process call to retrieve network info."""
        cctxt = self.client.prepare()
        network = cctxt.call(self.context, 'get_network_info',
                             network_id=network_id, host=self.host)
        if network:
            return dhcp.NetModel(self.use_namespaces, network)

    def create_dhcp_port(self, port):
        """Make a remote process call to create the dhcp port."""
        cctxt = self.client.prepare(version='1.1')
        port = cctxt.call(self.context, 'create_dhcp_port',
                          port=port, host=self.host)
        if port:
            return dhcp.DictModel(port)

    def update_dhcp_port(self, port_id, port):
        """Make a remote process call to update the dhcp port."""
        cctxt = self.client.prepare(version='1.1')
        port = cctxt.call(self.context, 'update_dhcp_port',
                          port_id=port_id, port=port, host=self.host)
        if port:
            return dhcp.DictModel(port)

    def release_dhcp_port(self, network_id, device_id):
        """Make a remote process call to release the dhcp port."""
        cctxt = self.client.prepare()
        return cctxt.call(self.context, 'release_dhcp_port',
                          network_id=network_id, device_id=device_id,
                          host=self.host)

    def release_port_fixed_ip(self, network_id, device_id, subnet_id):
        """Make a remote process call to release a fixed_ip on the port."""
        cctxt = self.client.prepare()
        return cctxt.call(self.context, 'release_port_fixed_ip',
                          network_id=network_id, subnet_id=subnet_id,
                          device_id=device_id, host=self.host)

```


### **NetworkCache类**


主要作为当前network信息的缓存


```
    def put_port(self, port):
        network = self.get_network_by_id(port.network_id)
        for index in range(len(network.ports)):
            if network.ports[index].id == port.id:
                network.ports[index] = port
                break
        else:
            network.ports.append(port)

        self.port_lookup[port.id] = network.id

    def remove_port(self, port):
        network = self.get_network_by_port_id(port.id)

        for index in range(len(network.ports)):
            if network.ports[index] == port:
                del network.ports[index]
                del self.port_lookup[port.id]
                break

    def get_port_by_id(self, port_id):
        network = self.get_network_by_port_id(port_id)
        if network:
            for port in network.ports:
                if port.id == port_id:
                    return port

    def get_state(self):
        net_ids = self.get_network_ids()
        num_nets = len(net_ids)
        num_subnets = 0
        num_ports = 0
        for net_id in net_ids:
            network = self.get_network_by_id(net_id)
            num_subnets += len(network.subnets)
            num_ports += len(network.ports)
        return {'networks': num_nets,
                'subnets': num_subnets,
                'ports': num_ports}


```





### **DhcpAgentWithStateReport**


该类继承自DhcpAgent，主要添加了状态汇报。 汇报状态主要是DhcpAgentWithStateReport初始化中指定了一个agent_rpc.PluginReportStateAPI(topics.PLUGIN)类作为状态汇报rpc消息的处理handler



```
 # 作为RPC的client端，topic类型为topics.PLUGIN，汇报RPC消息
 self.state_rpc = agent_rpc.PluginReportStateAPI(topics.PLUGIN)

```

这些代码会让_report_state()定期执行来汇报自身状态。

```
        report_interval = self.conf.AGENT.report_interval
        if report_interval:
            self.heartbeat = loopingcall.FixedIntervalLoopingCall(
                self._report_state)
            self.heartbeat.start(interval=report_interval)
```


`_report_state()`具体实现方式

```
    def _report_state(self):
        try:
            self.agent_state.get('configurations').update(
                self.cache.get_state())
            ctx = context.get_admin_context_without_session()
            agent_status = self.state_rpc.report_state(
                ctx, self.agent_state, True)
            if agent_status == constants.AGENT_REVIVED:
                LOG.info(_LI("Agent has just been revived. "
                             "Scheduling full sync"))
                self.schedule_resync("Agent has just been revived")
        except AttributeError:
            # This means the server does not support report_state
            LOG.warn(_LW("Neutron server does not support state report."
                         " State report for this agent will be disabled."))
            self.heartbeat.stop()
            self.run()
            return
        except Exception:
            LOG.exception(_LE("Failed reporting state!"))
            return
        if self.agent_state.pop('start_flag', None):
            self.run()


```












