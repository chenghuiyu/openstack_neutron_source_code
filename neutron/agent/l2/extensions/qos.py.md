## **qos.py**
------------------------

### **QosAgentDriver类**

主要定义了QoS agent driver的抽象接口，并定义了`create`、`update`、`delete`方法，这个方法需要一定的参数，包括具体的action、port和qos_policy。对于QosAgent来说
主要通过实现QoS agent driver的抽象接口来调用QoS agent driver实现相应的QOS policy。


```
	# 定义抽象的接口
    @abc.abstractmethod
    def initialize(self):
        """Perform QoS agent driver initialization.
        """

    def create(self, port, qos_policy):
        """Apply QoS rules on port for the first time.

        :param port: port object.
        :param qos_policy: the QoS policy to be applied on port.
        """
        self._handle_update_create_rules('create', port, qos_policy)

    def update(self, port, qos_policy):
        """Apply QoS rules on port.

        :param port: port object.
        :param qos_policy: the QoS policy to be applied on port.
        """
        self._handle_update_create_rules('update', port, qos_policy)

    def delete(self, port, qos_policy=None):
        """Remove QoS rules from port.

        :param port: port object.
        :param qos_policy: the QoS policy to be removed from port.
        """
        if qos_policy is None:
            rule_types = self.SUPPORTED_RULES
        else:
            rule_types = set(
                [rule.rule_type
                 for rule in self._iterate_rules(qos_policy.rules)])

        for rule_type in rule_types:
            self._handle_rule_delete(port, rule_type)


```

下面具体来看`_handle_update_create_rules()`方法，这是一个内部类的方法，主要就是更新一下创建的policy规则。

```
    def _handle_update_create_rules(self, action, port, qos_policy):
        for rule in self._iterate_rules(qos_policy.rules):
            if rule.should_apply_to_port(port):
		# handler的rule类型和qos driver定义保持一致，包括create_、update_、delete_
                handler_name = "".join((action, "_", rule.rule_type))
		# Get a named attribute from an object; getattr(x, 'y') is equivalent to x.y.
                handler = getattr(self, handler_name)
                handler(port, rule)
            else:
                LOG.debug("Port %(port)s excluded from QoS rule %(rule)s",
                          {'port': port, 'rule': rule.id})


```


`_iterate_rules()`是一个有关rule的generator


```
    def _iterate_rules(self, rules):
        for rule in rules:
            rule_type = rule.rule_type
	# qos driver应该定义好可以支持的rules类型，包括create_、update_、delete_等，并与handler保持一致，这里使用yield来定义有关rule的generator。
            if rule_type in self.SUPPORTED_RULES:
                yield rule
            else:
                LOG.warning(_LW('Unsupported QoS rule type for %(rule_id)s: '
                                '%(rule_type)s; skipping'),
                            {'rule_id': rule.id, 'rule_type': rule_type})


```


### **PortPolicyMap类**

主要实现port和policy之间的映射，比如将port绑定到新的policy，并返回该port的所有policy，或者将port和之前的policy分离，并删除有关该policy的所有数据。


```
    def set_port_policy(self, port, policy):
		# 将port和policy进行绑定
        """Attach a port to policy and return any previous policy on port."""
        port_id = port['port_id']
        old_policy = self.get_port_policy(port)
        self.known_policies[policy.id] = policy
        self.port_policies[port_id] = policy.id
		# port和policy进行绑定
        self.qos_policy_ports[policy.id][port_id] = port
        if old_policy and old_policy.id != policy.id:
            del self.qos_policy_ports[old_policy.id][port_id]
        return old_policy

```



### **QosAgentExtension类**

继承父类`agent_extension.AgentCoreResourceExtension`，并覆盖了原有的一些方法。

```
SUPPORTED_RESOURCES = [resources.QOS_POLICY]
```


```
    def initialize(self, connection, driver_type):
        """Perform Agent Extension initialization.

        """
	# 初始化qos_driver的RPC消息通信，实现了server端的CallBack
        self.resource_rpc = resources_rpc.ResourcesPullRpcApi()
	# 调用NeutronManager的load_class_for_provider方法
	# namespace是'neutron.qos.agent_drivers'，plugin_provider就是driver_type
        self.qos_driver = manager.NeutronManager.load_class_for_provider(
            'neutron.qos.agent_drivers', driver_type)()
        self.qos_driver.initialize()
	# 映射port和policy之间的关系
        self.policy_map = PortPolicyMap()
	# 定义resources.QOS_POLICY的callback，即_handle_notification
        registry.subscribe(self._handle_notification, resources.QOS_POLICY)
	# 定义consumer的RPC通信连接，这里应该就是server端
        self._register_rpc_consumers(connection)

```

这里其实还存在一个问题，就是如何根据resource_type生成具体的callback方法，上面的resource_type对应的就是resources.QOS_POLICY，而`neutron.api.rpc.callbacks.consumer`中的subscribe()方法如下所示：

```
def _get_manager():
    return resource_manager.ConsumerResourceCallbacksManager()


def subscribe(callback, resource_type):
    _get_manager().register(callback, resource_type)

```
`neutron.api.rpc.callbacks.comsumer`是继承了父类`neutron.api.rpc.callbacks`，而subscribe()也是调用了`ConsumerResourceCallbacksManager()`的类方法，`ConsumerResourceCallbacksManager`类继承了父类`ResourceCallbacksManager`，父类的`register`方法如下：

```
    def register(self, callback, resource_type):
        """Register a callback for a resource type.

        :param callback: the callback. It must raise or return NeutronObject.
        :param resource_type: must be a valid resource type.
        """
        LOG.debug("Registering callback for %s", resource_type)
        _validate_resource_type(resource_type)
        self._add_callback(callback, resource_type)

```

但是`_add_callback()`是一个抽象接口，具体实现是在`ConsumerResourceCallbacksManager`类中，

```
    def _add_callback(self, callback, resource_type):
        self._callbacks[resource_type].add(callback)

```
即生成一个带有resource_type和callback的set集合，到此就已经注册好了一个指定resource_type类型的callback方法。







`handle_port()`方法的具体实现

```
    @lockutils.synchronized('qos-port')
    def handle_port(self, context, port):
        """Handle agent QoS extension for port.

        This method applies a new policy to a port using the QoS driver.
        Update events are handled in _handle_notification.
        """
        port_id = port['port_id']
        port_qos_policy_id = port.get('qos_policy_id')
        network_qos_policy_id = port.get('network_qos_policy_id')
        qos_policy_id = port_qos_policy_id or network_qos_policy_id
        if qos_policy_id is None:
            self._process_reset_port(port)
            return

        if not self.policy_map.has_policy_changed(port, qos_policy_id):
            return

        qos_policy = self.resource_rpc.pull(
            context, resources.QOS_POLICY, qos_policy_id)
        if qos_policy is None:
            LOG.info(_LI("QoS policy %(qos_policy_id)s applied to port "
                         "%(port_id)s is not available on server, "
                         "it has been deleted. Skipping."),
                     {'qos_policy_id': qos_policy_id, 'port_id': port_id})
            self._process_reset_port(port)
        else:
            old_qos_policy = self.policy_map.set_port_policy(port, qos_policy)
            if old_qos_policy:
                self.qos_driver.delete(port, old_qos_policy)
                self.qos_driver.update(port, qos_policy)
            else:
                self.qos_driver.create(port, qos_policy)

```



> python学习小结：有关yield的用法。当一个函数包含了yield，那么表示该函数已经是一个generator。在Python中，这种一边循环一边计算的机制，称为生成器（Generator），generator和函数的执行流程不一样。函数是顺序执行，遇到return语句或者最后一行函数语句就返回。而变成generator的函数，在每次调用next()的时候执行，遇到yield语句返回，再次执行时从上次返回的yield语句处继续执行。
