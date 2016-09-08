# neutron中的QOS实现源码-**extension中的实现**

--------------------------------------------------------------

主要实现了基类`QoSPluginBase`,里面主要定义了和qos相关的抽象方法：

- `get_policy()`
- `get_policies()`
- `create_policy()`
- `update_policy()`
- `delete_policy()`
- `get_policy_bandwidth_limit_rule()`
- `get_policy_bandwidth_limit_rules()`
- `create_policy_bandwidth_limit_rule()`
- `update_policy_bandwidth_limit_rule()`  
- `delete_policy_bandwidth_limit_rule()`  
- `get_rule_types()`







