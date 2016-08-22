## **service.py**
---------

### **初始配置**
代码开始定义了服务相关的配置信息，具体包括：

- periodic_interval：周期性任务之间的间隔大小，默认为40秒；
- api_workers：处理服务的API worker数量，默认为CPU数；
- rpc_workers：处理服务的RPC worker数量，默认为1个
- periodic_fuzzy_delay：开启periodic task scheduler所用的延迟时间，默认为5秒。

定义好配置参数后，写入配置文件里去：

```
CONF = cfg.CONF
CONF.register_opts(service_opts)
```



### **WegiService类函数**

用于服务的WSGI的基函数

理解为将传入的API 请求转换为wsgi服务，具体就是传入参数`app_name`,调用`_run_wsgi`函数来得到`wsgi_app`。

```
def start(self):
	self.wsgi_app = _run_wsgi(self.app_name)
```




### **NeutronApiService类函数**

使用`@classmethod`直接使用类名.方法名调用方法，得到neutron API具体的service。

```
service = cls(app_name)
```


### **RpcWorker类函数**

主要包含了start()、wait()、stop()和reset()方法，这个类需要传入一个NeutronWorker的实例，如果一个plugin需要和neutron的数据库进行同步，并且这个动作只执行一次，那么就不需要在每一个API worker都执行，只需要使用`get_workers`方法来得到NeutronWorkers的多个实例就可以了。

```
        class MyPlugin(...):
            def get_workers(self):
                return [MyPluginWorker()]

        class MyPluginWorker(NeutronWorker):
            def start(self):
                super(MyPluginWorker, self).start()
                do_sync()
```

这里RpcWorker类的实例的功能类似于这个功能。

### **Service类**

在openstack中，各个服务的组件都以Service类的方式来进行交互，在neutron中，Service类在多个地方出现，包括`neutron.service.Service`类，`neutron.common.rpc.Service`类和`oslo_service.Service`类。通常来说，`oslo_service.Service`类是所有service的父类，而`neutron.common.rpc.Service`类继承于`oslo_service.Service`类，`neutron.service.Service`类继承于`neutron.common.rpc.Service`类。

Server类主要包括以下几个方法函数：

```
    def __init__(self, host, binary, topic, manager, report_interval=None,
                 periodic_interval=None, periodic_fuzzy_delay=None,
                 *args, **kwargs):

        self.binary = binary
        self.manager_class_name = manager
        manager_class = importutils.import_class(self.manager_class_name)
        self.manager = manager_class(host=host, *args, **kwargs)
        self.report_interval = report_interval
        self.periodic_interval = periodic_interval
        self.periodic_fuzzy_delay = periodic_fuzzy_delay
        self.saved_args, self.saved_kwargs = args, kwargs
        self.timers = []
        super(Service, self).__init__(host, topic, manager=self.manager)
```




#### `start()`

```
    def start(self):
        """调用Neutron.manager的init_host()方法，对一个单独的服务进行初始化 """
        self.manager.init_host()
        super(Service, self).start()
        """主要将server端的状态周期性的进行报告，调用loopingcall的方法进行汇报 """
        if self.report_interval:
            pulse = loopingcall.FixedIntervalLoopingCall(self.report_state)
            pulse.start(interval=self.report_interval,
                        initial_delay=self.report_interval)
            self.timers.append(pulse)
        """读取配置文件的periodic_interval和periodic_fuzzy_delay参数 """
        if self.periodic_interval:
            if self.periodic_fuzzy_delay:
		"""生成一个随机的initial_delay"""
                initial_delay = random.randint(0, self.periodic_fuzzy_delay)
            else:
                initial_delay = None
		"""调用loopingcall类方法生产固定间隔的looping call消息发送方式"""
            periodic = loopingcall.FixedIntervalLoopingCall(
                self.periodic_tasks)
            periodic.start(interval=self.periodic_interval,
                           initial_delay=initial_delay)
            self.timers.append(periodic)
        self.manager.after_start()

```



#### `creat()`

```
    def create(cls, host=None, binary=None, topic=None, manager=None,
               report_interval=None, periodic_interval=None,
               periodic_fuzzy_delay=None):
        """Instantiates class and passes back application object.
		这些参数都是从配置文件中读取的
        :param host: defaults to CONF.host
        :param binary: defaults to basename of executable
        :param topic: defaults to bin_name - 'neutron-' part
        :param manager: defaults to CONF.<topic>_manager
        :param report_interval: defaults to CONF.report_interval
        :param periodic_interval: defaults to CONF.periodic_interval
        :param periodic_fuzzy_delay: defaults to CONF.periodic_fuzzy_delay

        """
        if not host:
            host = CONF.host
        if not binary:
            binary = os.path.basename(inspect.stack()[-1][1])
        if not topic:
            topic = binary.rpartition('neutron-')[2]
            topic = topic.replace("-", "_")
        if not manager:
            manager = CONF.get('%s_manager' % topic, None)
        if report_interval is None:
            report_interval = CONF.report_interval
        if periodic_interval is None:
            periodic_interval = CONF.periodic_interval
        if periodic_fuzzy_delay is None:
            periodic_fuzzy_delay = CONF.periodic_fuzzy_delay
		"""如果配置文件中都有上述参数，那么初始化自身类cls """
        service_obj = cls(host, binary, topic, manager,
                          report_interval=report_interval,
                          periodic_interval=periodic_interval,
                          periodic_fuzzy_delay=periodic_fuzzy_delay)

        return service_obj

```


#### `kill()`

#### `stop()`

#### `wait()`

#### `reset()`

#### `periodic_tasks()`

#### `report_state()`

z总之，一方面运行一个基于topic的manager来监听消息队列，另外一方面，可以在manager上跑周期性的任务，并汇报状态。


### start_plugin_workers方法

```
    for plugin in manager.NeutronManager.get_unique_service_plugins():
        # TODO(twilson) Instead of defaulting here, come up with a good way to
        # share a common get_workers default between NeutronPluginBaseV2 and
        # ServicePluginBase
        for plugin_worker in getattr(plugin, 'get_workers', tuple)():
            launcher = common_service.ProcessLauncher(cfg.CONF)
            launcher.launch_service(plugin_worker)
            launchers.append(launcher)
```
从元数组plugin中利用`get_workers`得到workers，调用`launch_server`得到相关的plugin workers对象。


### serve_rpc方法
这个方法主要用来生成RPC的服务端，通过调用plugin的方法来实现

```
        rpc = RpcWorker(plugin)

        # dispose the whole pool before os.fork, otherwise there will
        # be shared DB connections in child processes which may cause
        # DB errors.
        LOG.debug('using launcher for rpc, workers=%s', cfg.CONF.rpc_workers)
        session.dispose()
        launcher = common_service.ProcessLauncher(cfg.CONF, wait_interval=1.0)
        launcher.launch_service(rpc, workers=cfg.CONF.rpc_workers)
        return launcher

```
'RpcWorker(plugin)'来调用'start'方法产生用来监听来自plugin的消息

```
    def start(self):
        super(RpcWorker, self).start()
        self._servers = self._plugin.start_rpc_listeners()
```

'start_rpc_listeners()'目前ml2可以实现相关的方法，在RabbitMQ中定义的topics为topics.PLUGIN的形式来接受agents那边过来的消息，有关plugin,agent和server之间的RPC调用后面会有专题进行讲述。

```
    def start_rpc_listeners(self):
        """Start the RPC loop to let the plugin communicate with agents."""
        self._setup_rpc()
        self.topic = topics.PLUGIN
        self.conn = n_rpc.create_connection(new=True)
        self.conn.create_consumer(self.topic, self.endpoints, fanout=False)
```

所以这个方法最终会返回server端的RPC worker来监听消息。








> python学习小结：一般来说，要使用某个类的方法，需要先实例化一个对象再调用方法。而使用@staticmethod或@classmethod，就可以不需要实例化，直接类名.方法名()来调用。这有利于组织代码，把某些应该属于某个类的函数给放到那个类里去，同时有利于命名空间的整洁。@classmethod也不需要self参数，但第一个参数需要是表示自身类的cls参数。如果在@staticmethod中要调用到这个类的一些属性方法，只能直接类名.属性名或类名.方法名。而@classmethod因为持有cls参数，可以来调用类的属性，类的方法，实例化对象等，避免硬编码。
