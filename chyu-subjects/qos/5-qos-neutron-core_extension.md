# neutron中的QOS实现源码-*core_extensions类*中的实现**

-------------------------------------------------------


`core_extensions类`中主要实现了一些核心的扩展方法


## **base.py**

主要定义了一个基类`CoreResourceExtension`，定义了两个抽象接口，`process_fields`和`extract_fields`，传入的参数包括network、port类型以及扩展域，对于qos来说，
这个扩展域可以理解为qos的policy。


```
"""资源类型主要包括network和port信息 """

NETWORK = 'network'
PORT = 'port'



CORE_RESOURCES = [NETWORK, PORT]


@six.add_metaclass(abc.ABCMeta)
class CoreResourceExtension(object):
	# 抽象方法，
    @abc.abstractmethod
    def process_fields(self, context, resource_type,
                       requested_resource, actual_resource):
        """Process extension fields.

        :param context: neutron api request context
        :param resource_type: core resource type (one of CORE_RESOURCES)
        :param requested_resource: resource dict that contains extension fields
        :param actual_resource: actual resource dict known to plugin
        """

    @abc.abstractmethod
    def extract_fields(self, resource_type, resource):
        """Extract extension fields.

        :param resource_type: core resource type (one of CORE_RESOURCES)
        :param resource: resource dict that contains extension fields
        """

```


## **qos.py**

现在来具体看qos是如何对抽象方法进行实现的，主要实现了`QosCoreResourceExtension`类，该类是继承基类`CoreResourceExtension`，里面具体实现了`process_fields`和`extract_fields`。


```
	# @property将定义的方法装饰为属性，直接进行调用
	"""
	判断qos的plugin是否已经加载，如果没有就调用NeutronManager的get_service_plugins()进行加载，
	常量plugin_constants.QOS用于遍历，返回结果
	"""
    @property
    def plugin_loaded(self):
        if not hasattr(self, '_plugin_loaded'):
            service_plugins = manager.NeutronManager.get_service_plugins()
            self._plugin_loaded = plugin_constants.QOS in service_plugins
        return self._plugin_loaded


```

下面来看两个内部方法，`_get_policy_obj`和`_update_port_policy`

对于`_get_policy_obj`来说，类`QosPolicy`位于`neutron.objects.qos.policy`中，通过传入的policy_id得到有关qos的policy对象
对于`_update_port_policy`来说，根据port的ID来找出对应qos的policy，`detach_port`方法直接将找出的`old_policy`进行删除；接着在需要更新policy的port上找到
qos_policy_id，然后获得policy对象，将更新policy的port信息加到数据库中去。

```
    def _get_policy_obj(self, context, policy_id):
        obj = policy_object.QosPolicy.get_by_id(context, policy_id)
        if obj is None:
            raise n_exc.QosPolicyNotFound(policy_id=policy_id)
        return obj

    def _update_port_policy(self, context, port, port_changes):
        old_policy = policy_object.QosPolicy.get_port_policy(
            context, port['id'])
        if old_policy:
            old_policy.detach_port(port['id'])

        qos_policy_id = port_changes.get(qos_consts.QOS_POLICY_ID)
        if qos_policy_id is not None:
            policy = self._get_policy_obj(context, qos_policy_id)
            policy.attach_port(port['id'])
        port[qos_consts.QOS_POLICY_ID] = qos_policy_id


```
`detach_port`方法在类`QosPolicy`进行实现，直接在数据库中删除有关old_qos_policy的数据。

```

    def detach_port(self, port_id):
        qos_db_api.delete_policy_port_binding(self._context,
                                              policy_id=self.id,
                                              port_id=port_id)

```
`attach_port`方法在类`QosPolicy`进行实现，直接在数据库中添加有关qos_policy的数据。

```

    def attach_port(self, port_id):
        qos_db_api.create_policy_port_binding(self._context,
                                              policy_id=self.id,
                                              port_id=port_id)

```


`_update_network_policy`方法主要实现network的policy的操作，实现原理和port一致。


```
    def _update_network_policy(self, context, network, network_changes):
        old_policy = policy_object.QosPolicy.get_network_policy(
            context, network['id'])
        if old_policy:
            old_policy.detach_network(network['id'])

        qos_policy_id = network_changes.get(qos_consts.QOS_POLICY_ID)
        if qos_policy_id is not None:
            policy = self._get_policy_obj(context, qos_policy_id)
            policy.attach_network(network['id'])
        network[qos_consts.QOS_POLICY_ID] = qos_policy_id


```

上述有关port和network的操作，就是对应的resource_type



既然是继承自`CoreResourceExtension`，就要实现里面的抽象接口。


```
    def process_fields(self, context, resource_type,
                       requested_resource, actual_resource):
        if (qos_consts.QOS_POLICY_ID in requested_resource and
            self.plugin_loaded):
            self._exec('_update_%s_policy' % resource_type, context,
                       {resource_type: actual_resource,
                        "%s_changes" % resource_type: requested_resource})

    def extract_fields(self, resource_type, resource):
        if not self.plugin_loaded:
            return {}
  
        binding = resource['qos_policy_binding']
        qos_policy_id = binding['policy_id'] if binding else None
        return {qos_consts.QOS_POLICY_ID: qos_policy_id}


```



`_exec`方法

```
    def _exec(self, method_name, context, kwargs):
        with db_api.autonested_transaction(context.session):
            return getattr(self, method_name)(context=context, **kwargs)
```

`db_api.autonested_transaction`是`neutron.db.api`中定义的方法，防止被传入的参数嵌套。

```
@contextlib.contextmanager
def autonested_transaction(sess):
    """This is a convenience method to not bother with 'nested' parameter."""
    try:
        session_context = sess.begin_nested()
    except exc.InvalidRequestError:
        session_context = sess.begin(subtransactions=True)
    finally:
        with session_context as tx:
            yield tx

```



