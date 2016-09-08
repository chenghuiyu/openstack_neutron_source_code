# neutron中的QOS实现源码-service中的实现

------------------------------------------------------------------

## **notification_drivers文件夹**

### **1、manager.py**

主要实现了`QosServiceNotificationDriverManager`类，首先进行初始化，得到qos的drivers实例


```
    def __init__(self):
        self.notification_drivers = []
        self._load_drivers(cfg.CONF.qos.notification_drivers)

```

根据传入的`notification_drivers`参数，得到实例instance，具体实现通过调用内部方法`_load_driver_instance()`

```
    def _load_drivers(self, notification_drivers):
        """Load all the instances of the configured QoS notification drivers

        :param notification_drivers: comma separated string
        """
        if not notification_drivers:
            raise SystemExit(_('A QoS driver must be specified'))
        LOG.debug("Loading QoS notification drivers: %s", notification_drivers)
        for notification_driver in notification_drivers:
            driver_ins = self._load_driver_instance(notification_driver)
            self.notification_drivers.append(driver_ins)

```

方法`_load_driver_instance()`调用类`NeutronManager`的`load_class_for_provider()`方法，对driver进行实例化。


```

    def _load_driver_instance(self, notification_driver):
        """Returns an instance of the configured QoS notification driver

        :returns: An instance of Driver for the QoS notification
        """
        mgr = manager.NeutronManager
        driver = mgr.load_class_for_provider(QOS_DRIVER_NAMESPACE,
                                             notification_driver)
        driver_instance = driver()
        LOG.info(
            _LI("Loading %(name)s (%(description)s) notification driver "
                "for QoS plugin"),
            {"name": notification_driver,
             "description": driver_instance.get_description()})
        return driver_instance

```

`load_class_for_provider()`方法中的实现过程，调用python中的类`stevedore`。根据namespace获得相应entry point组，然后根据plugin_provider调用相应的plugin插件

```
            mgr = driver.DriverManager(namespace, plugin_provider)
            plugin_class = mgr.driver

```



得到drivers的实例后就可以对其进行具体的操作了，包括：`update_policy()`、`delete_policy()`和`create_policy()`

```
    def update_policy(self, context, qos_policy):
        for driver in self.notification_drivers:
            driver.update_policy(context, qos_policy)

```




```
    def delete_policy(self, context, qos_policy):
        for driver in self.notification_drivers:
            driver.delete_policy(context, qos_policy)
```


```
    def create_policy(self, context, qos_policy):
        for driver in self.notification_drivers:
            driver.create_policy(context, qos_policy)

```

### **2、message_queue.py**



主要实现了类`RpcQosServiceNotificationDriver`，主要用来实现qos用于消息队列的driver。继承父类`neutron.services.qos.notification_drivers.QosServiceNotificationDriverBase`。


初始化实现了`ResourcesPushRpcApi()`类的实例化

```
        self.notification_api = resources_rpc.ResourcesPushRpcApi()
		registry.provide(_get_qos_policy_cb, resources.QOS_POLICY)

```

继承父类的`push()`方法

```
    def update_policy(self, context, policy):
        self.notification_api.push(context, policy, events.UPDATED)

```

`push(self, context, resource, event_type)`实现方法，`ResourcesPushRpcApi()`类主要将对象的更新状态信息push给receiver端的agent，通过消息队列的广播topic形式发送。

最终也是以一种cast的形式广播出去，这种并不需要接收端的反馈，push出去就不需要等待接收反馈了。

```
        cctxt = self._prepare_object_fanout_context(resource)
        dehydrated_resource = resource.obj_to_primitive()
        cctxt.cast(context, 'push',
                   resource=dehydrated_resource,
                   event_type=event_type)

```






### **3、qos_base.py**

主要包括类`QosServiceNotificationDriverBase`，里面全是抽象接口，具体需要继承的子类来实现


- `get_description`
- `create_policy`
- `update_policy`
- `delete_policy`















## **qos_consts.py**

配置好关键字和参数，进行全局调用，`RULE_TYPE_BANDWIDTH_LIMIT`，`VALID_RULE_TYPES`，`QOS_POLICY_ID`

```
RULE_TYPE_BANDWIDTH_LIMIT = 'bandwidth_limit'
VALID_RULE_TYPES = [RULE_TYPE_BANDWIDTH_LIMIT]

QOS_POLICY_ID = 'qos_policy_id'

```

## **qos_plugin.py**

主要定义了类`QosPlugin`，该类主要继承父类`neutron.extensions.QoSPluginBase`，实现了neutron QOS service端的plugin，为network和port提供qos参数

该类实现了neutron QOS 服务的plugin，提供network和port上的qos流量控制的plugin。

首先进行了初始化，实例化类`QosServiceNotificationDriverManager()`，再进行后续的操作。

```
        self.notification_driver_manager = (
            driver_mgr.QosServiceNotificationDriverManager())

```

### **policy的方法实现**


方法`create_policy()`、`update_policy()`、`delete_policy()`、`get_policy()`,根据传入的参数`policy`或者`policy_id`进行相应的操作。

```
        policy = policy_object.QosPolicy(context, **policy['policy'])
        policy.create()
        self.notification_driver_manager.create_policy(context, policy)

```


这里主要是实例化`neutron.objects.qos.policy.QosPolicy()`类，并调用`create()`、`update()`、`delete()`、`get_by_id()`方法，即是增删改查。具体来说就是操作底层的数据库，进行增删改查。

`neutron.objects`主要是和数据库层打交道的。

在neutron.objects.qos.policy.QosPolicy()`类中，`create()`方法实例化了session层的映射，将对象和数据库中的表格映射起来。

```
    def create(self):
        with db_api.autonested_transaction(self._context.session):
            super(QosPolicy, self).create()
            self.reload_rules()

```

```
    def delete(self):
        models = (
            ('network', self.network_binding_model),
            ('port', self.port_binding_model)
        )
        with db_api.autonested_transaction(self._context.session):
            for object_type, model in models:
                binding_db_obj = db_api.get_object(self._context, model,
                                                   policy_id=self.id)
                if binding_db_obj:
                    raise exceptions.QosPolicyInUse(
                        policy_id=self.id,
                        object_type=object_type,
                        object_id=binding_db_obj['%s_id' % object_type])

            super(QosPolicy, self).delete()

```

```
    def get_by_id(cls, context, id):
        admin_context = context.elevated()
        with db_api.autonested_transaction(admin_context.session):
            policy_obj = super(QosPolicy, cls).get_by_id(admin_context, id)
            if (not policy_obj or
                not cls._is_policy_accessible(context, policy_obj)):
                return

            policy_obj.reload_rules()
            return policy_obj

```


### **rule的方法实现**

分析完policy的配置，下面来看rule的具体操作，实现方法包括：`create_policy_bandwidth_limit_rule()`、`update_policy_bandwidth_limit_rule()`、`delete_policy_bandwidth_limit_rule()`、`get_policy_bandwidth_limit_rule()`、`get_policy_bandwidth_limit_rules()`


#### `create_policy_bandwidth_limit_rule()`方法

先实例化session的映射对象，将数据库的表和object进行映射，然后根据policy_id以及传入的参数`bandwidth_limit_rule`，先从`neutron.object`中实例化session,然后在数据库中进行表的添加。

数据库添加完成后，在对policy进行update一下，保证rule的实时性。



```
            policy = self._get_policy_obj(context, policy_id)
            rule = rule_object.QosBandwidthLimitRule(
                context, qos_policy_id=policy_id,
                **bandwidth_limit_rule['bandwidth_limit_rule'])
            rule.create()
            policy.reload_rules()
        self.notification_driver_manager.update_policy(context, policy)


```


`neutron.objects.qos.QosRule`的子类`QosBandwidthLimitRule`中实现的步骤，可见直接调用`neutorn.db.qos.modles`操作数据库

```
    db_model = qos_db_model.QosBandwidthLimitRule
```


`neutorn.db.qos.modles`中的类`QosBandwidthLimitRule`实现过程，


```
    qos_policy_id = sa.Column(sa.String(36),
                              sa.ForeignKey('qos_policies.id',
                                            ondelete='CASCADE'),
                              nullable=False,
                              unique=True)
    max_kbps = sa.Column(sa.Integer)
    max_burst_kbps = sa.Column(sa.Integer)

```

这就是在数据库中对表数据的实际操作。操作结束再对policy更新一下rule


`update_policy_bandwidth_limit_rule()`和`delete_policy_bandwidth_limit_rule()`实现过程类似，也是对后台数据库进行操作然后reload_rules一下。



#### `get_policy_bandwidth_limit_rule()`

调用方式也比较类似，主要通过`neutorn.objects`来对session进行映射实例化，然后从后台的数据库中进行查询

```
            self._get_policy_obj(context, policy_id)
            rule = rule_object.QosBandwidthLimitRule.get_by_id(
                context, rule_id)

```


