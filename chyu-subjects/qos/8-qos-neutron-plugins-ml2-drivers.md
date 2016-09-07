# neutron中的QOS实现源码-ml2的drivers中的实现

------------------------------------------------------------

这里主要以openvswitch中的drivers的qos实现为例来进行说明

## **QosOVSAgentDriver类**

从`qos_consts.RULE_TYPE_BANDWIDTH_LIMIT`得到一个元数组`SUPPORTED_RULES`

```
    SUPPORTED_RULES = (
        mech_openvswitch.OpenvswitchMechanismDriver.supported_qos_rule_types)
```

接着对ovs进行初始化，`OVSBridge类`主要是继承父类`BaseOVS`实现`neutron.agent.ovsdb.api`中的抽象方法，完成br-int的初始化

```
    def initialize(self):
        self.br_int = ovs_lib.OVSBridge(self.br_int_name)

```

`update_bandwidth_limit()`方法主要根据port信息，以及qos rule规则，对网络的出口流量带宽进行限制，但从代码上来看，真正实现的是底层的ovs，neutron只是利用drivers进行调用。




```
    def update_bandwidth_limit(self, port, rule):
        port_name = port['vif_port'].port_name
        max_kbps = rule.max_kbps
        max_burst_kbps = rule.max_burst_kbps

        self.br_int.create_egress_bw_limit_for_port(port_name,
                                                    max_kbps,
                                                    max_burst_kbps)

```

`create_egress_bw_limit_for_port()`方法主要在`OVSBridge类`实现，具体也是调用内部方法`_set_egress_bw_limit_for_port()`实现。

```
    def create_egress_bw_limit_for_port(self, port_name, max_kbps,
                                        max_burst_kbps):
        self._set_egress_bw_limit_for_port(
            port_name, max_kbps, max_burst_kbps)

```

`_set_egress_bw_limit_for_port()`方法传入三个参数：`port_name`、`max_kbps`、`max_burst_kbps`

`ovsdb.transaction()`是`neutron.agent.ovsdb.api`中的抽象接口，具体实现是在`impl_vsctl.py`中实现抽象接口。相当于API接受命令请求，ovs运行底层命令对流量进行设置。

```
    def _set_egress_bw_limit_for_port(self, port_name, max_kbps,
                                      max_burst_kbps):
        with self.ovsdb.transaction(check_error=True) as txn:
            txn.add(self.ovsdb.db_set('Interface', port_name,
                                      ('ingress_policing_rate', max_kbps)))
            txn.add(self.ovsdb.db_set('Interface', port_name,
                                      ('ingress_policing_burst',
                                       max_burst_kbps)))
```
`db_set()`里面有两张表，table就是ovs里面维护的表，而record就是name-uuid的表，col_values是(column, value)的元数据

```
    def db_set(self, table, record, *col_values):
        args = [table, record]
        args += _set_colval_args(*col_values)
        return BaseCommand(self.context, 'set', args=args)

```


`BaseCommand`类来进行底层的映射操作

```
class BaseCommand(ovsdb.Command):
    def __init__(self, context, cmd, opts=None, args=None):
        self.context = context
        self.cmd = cmd
        self.opts = [] if opts is None else opts
        self.args = [] if args is None else args

    def execute(self, check_error=False, log_errors=True):
        with Transaction(self.context, check_error=check_error,
                         log_errors=log_errors) as txn:
            txn.add(self)
        return self.result

    def vsctl_args(self):
        return itertools.chain(('--',), self.opts, (self.cmd,), self.args)

```


## **OpenvswitchMechanismDriver类**


该类实现的方法比较简单，但是重点在于掌握里面的思想。

`OpenvswitchMechanismDriver`将ml2 plugin与openvswitch L2 agent集成起来，绑定在该driver上的port上面，需要有openvswitch L2 agent运行。agent至少需要和port的network段进行连接。


```
    def get_allowed_network_types(self, agent):
        return (agent['configurations'].get('tunnel_types', []) +
                [p_constants.TYPE_LOCAL, p_constants.TYPE_FLAT,
                 p_constants.TYPE_VLAN])

```
