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












### **3、qos_base.py**

















## **qos_consts.py**

配置好关键字和参数，进行全局调用，`RULE_TYPE_BANDWIDTH_LIMIT`，`VALID_RULE_TYPES`，`QOS_POLICY_ID`

```
RULE_TYPE_BANDWIDTH_LIMIT = 'bandwidth_limit'
VALID_RULE_TYPES = [RULE_TYPE_BANDWIDTH_LIMIT]

QOS_POLICY_ID = 'qos_policy_id'

```

## **qos_plugin.py**

主要定义了类`QosPlugin`，该类主要继承父类`neutron.extensions.QoSPluginBase`，实现了neutron QOS service端的plugin，为network和port提供qos参数









