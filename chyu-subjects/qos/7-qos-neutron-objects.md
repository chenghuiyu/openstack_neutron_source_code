# neutron中的QOS实现源码-**objects中的实现**

--------------------------------------------------

## **policy.py**




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
