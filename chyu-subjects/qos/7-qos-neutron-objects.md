# neutron中的QOS实现源码-objects中的实现

--------------------------------------------------

## **policy.py**

实现了对qos policy的增删改查操作，里面涉及到数据库中的操作。

### `QosPolicy类`


`QosPolicy`类继承父类`neutron.objects.NeutronDbObject`，先实例化数据库对象，后面会对resource_type进行CURD操作，包括`port_binding_model`、`network_binding_model`等。


```
    db_model = qos_db_model.QosPolicy
    port_binding_model = qos_db_model.QosPortPolicyBinding
    network_binding_model = qos_db_model.QosNetworkPolicyBinding

```


`to_dict`返回便于遍历的dict列表

```
    def to_dict(self):
        dict_ = super(QosPolicy, self).to_dict()
        if 'rules' in dict_:
            dict_['rules'] = [rule.to_dict() for rule in dict_['rules']]
        return dict_

```

父类`neutron.objects.NeutronDbObject`的实现方法

```
    def to_dict(self):
        return dict(self.items())

```





方法`get_by_id`可以直接进行引用，根据ID从数据库中查到`policy_obj`，调用`reload_rules()`后返回。

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


方法`get_objects()`

```
    def get_objects(cls, context, **kwargs):
        admin_context = context.elevated()
        with db_api.autonested_transaction(admin_context.session):
            objs = super(QosPolicy, cls).get_objects(admin_context,
                                                     **kwargs)
            result = []
            for obj in objs:
                if not cls._is_policy_accessible(context, obj):
                    continue
                obj.reload_rules()
                result.append(obj)
            return result

```



方法`get_network_policy()`

```
    def get_network_policy(cls, context, network_id):
        return cls._get_object_policy(context, cls.network_binding_model,
                                      network_id=network_id)
```


方法`get_port_policy()`

```
    def get_port_policy(cls, context, port_id):
        return cls._get_object_policy(context, cls.port_binding_model,
                                      port_id=port_id)


```


方法`create()`

```
    def create(self):
        with db_api.autonested_transaction(self._context.session):
            super(QosPolicy, self).create()
            self.reload_rules()

```


方法`delete`

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



方法`attach_network`、`attach_port`、`detach_network`、`detach_port`调用数据库进行增删改查操作。

```
    def attach_network(self, network_id):
        qos_db_api.create_policy_network_binding(self._context,
                                                 policy_id=self.id,
                                                 network_id=network_id)

    def attach_port(self, port_id):
        qos_db_api.create_policy_port_binding(self._context,
                                              policy_id=self.id,
                                              port_id=port_id)

    def detach_network(self, network_id):
        qos_db_api.delete_policy_network_binding(self._context,
                                                 policy_id=self.id,
                                                 network_id=network_id)

    def detach_port(self, port_id):
        qos_db_api.delete_policy_port_binding(self._context,
                                              policy_id=self.id,
                                              port_id=port_id)

```





## **rule.py**


#### `QosRule`类

主要继承自`neutron.objects.NeutronDbObject`父类

```
def should_apply_to_port(self, port):
        """Check whether a rule can be applied to a specific port.

        This function has the logic to decide whether a rule should
        be applied to a port or not, depending on the source of the
        policy (network, or port). Eventually rules could override
        this method, or we could make it abstract to allow different
        rule behaviour.
        """
        is_network_rule = self.qos_policy_id != port[qos_consts.QOS_POLICY_ID]
        is_network_device_port = any(port['device_owner'].startswith(prefix)
                                     for prefix
                                     in constants.DEVICE_OWNER_PREFIXES)

        return not (is_network_rule and is_network_device_port)

```
`should_apply_to_port`返回应用于network的port的rule结果

#### `QosBandwidthLimitRule`类

继承父类`QosRule`

得到了一个数据库rule对象
定义好枚举结构，生成rule_type

```
    db_model = qos_db_model.QosBandwidthLimitRule

    fields = {
        'max_kbps': obj_fields.IntegerField(nullable=True),
        'max_burst_kbps': obj_fields.IntegerField(nullable=True)
    }

    rule_type = qos_consts.RULE_TYPE_BANDWIDTH_LIMIT

```


## **rule_type.py**


主要实现了`QosRuleType`类和`RuleTypeField`类。


#### `RuleTypeField`类

继承了父类`oslo_versionedobjects.fields.BaseEnumField`，定义fields的key-filed的键值对

#### `QosRuleType`类 

继承父类`neutron.objects.NeutronObject`，循环遍历得到rule_types

```
    def get_objects(cls, **kwargs):
        cls.validate_filters(**kwargs)
        core_plugin = manager.NeutronManager.get_plugin()
        return [cls(type=type_)
                for type_ in core_plugin.supported_qos_rule_types]

```

#### `get_rules`方法

接收`neutron.db.api`的数据库API请求，使用session将对象和数据库表形成映射

```
def get_rules(context, qos_policy_id):
    all_rules = []
    with db_api.autonested_transaction(context.session):
        for rule_type in qos_consts.VALID_RULE_TYPES:
            rule_cls_name = 'Qos%sRule' % utils.camelize(rule_type)
            rule_cls = getattr(sys.modules[__name__], rule_cls_name)

            rules = rule_cls.get_objects(context, qos_policy_id=qos_policy_id)
            all_rules.extend(rules)
    return all_rules

```



