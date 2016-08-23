## **ovs_lib.py**

主要通过shell执行各种ovs-vsctl操作。

### **BaseOVS类**

操作OVS命令的基类，这个类包含了多个方法进行网桥的操作，这个方法的实现主要实现了`neutron.agent.ovsdb.api`类的抽象方法来对网桥进行操作。


```

    # 添加网桥，constants是在`neutron.plugins.ml2.drivers.openvswitch.agent.common.constants类`中实现的
	def add_bridge(self, bridge_name,
                   datapath_type=constants.OVS_DATAPATH_SYSTEM):

        self.ovsdb.add_br(bridge_name,
                          datapath_type).execute()
        br = OVSBridge(bridge_name)
        # Don't return until vswitchd sets up the internal port'
        br.get_port_ofport(bridge_name)
        return br
	# 删除网桥
    def delete_bridge(self, bridge_name):
        self.ovsdb.del_br(bridge_name).execute()

    # 网桥是否存在
    def bridge_exists(self, bridge_name):
        return self.ovsdb.br_exists(bridge_name).execute()
	# 端口是否存在
    def port_exists(self, port_name):
        cmd = self.ovsdb.db_get('Port', port_name, 'name')
        return bool(cmd.execute(check_error=False, log_errors=False))
	# 网桥接口
    def get_bridge_for_iface(self, iface):
        return self.ovsdb.iface_to_br(iface).execute()
	# 获取网桥
    def get_bridges(self):
        return self.ovsdb.list_br().execute(check_error=True)
	# 得到网桥ID
    def get_bridge_external_bridge_id(self, bridge):
        return self.ovsdb.br_get_external_id(bridge, 'bridge-id').execute()
	# 数据库配置
    def set_db_attribute(self, table_name, record, column, value,
                         check_error=False, log_errors=True):
        self.ovsdb.db_set(table_name, record, (column, value)).execute(
            check_error=check_error, log_errors=log_errors)
	# 清除数据库属性信息
    def clear_db_attribute(self, table_name, record, column):
        self.ovsdb.db_clear(table_name, record, column).execute()
	# 得到数据库信息
    def db_get_val(self, table, record, column, check_error=False,
                   log_errors=True):
        return self.ovsdb.db_get(table, record, column).execute(
            check_error=check_error, log_errors=log_errors)

```


### **OVSBridge类**


是继承自BaseOVS的子类，进一步丰富了父类的方法，包括端口port，tunnel的操作，也是实现了`neutron.agent.ovsdb.api`类的抽象方法。

```
	# 创建ofport
    def create(self, secure_mode=False):
        with self.ovsdb.transaction() as txn:
            txn.add(
                self.ovsdb.add_br(self.br_name,
                datapath_type=self.datapath_type))
            if secure_mode:
                txn.add(self.ovsdb.set_fail_mode(self.br_name,
                                                 FAILMODE_SECURE))
        # Don't return until vswitchd sets up the internal port'
        self.get_port_ofport(self.br_name)

```


这个地方比较巧妙，在调用create方法的时候，只有vswitchd这个进程在数据库中设置好port信息后，才进行return，创建方法返回的都是从数据库中读到的创建之后的port信息。


```
	# 增加ofport
    def add_port(self, port_name, *interface_attr_tuples):
        with self.ovsdb.transaction() as txn:
            txn.add(self.ovsdb.add_port(self.br_name, port_name))
            if interface_attr_tuples:
                txn.add(self.ovsdb.db_set('Interface', port_name,
                                          *interface_attr_tuples))
        return self.get_port_ofport(port_name)

```


为了说明add_port的具体的操作，需要从两个步骤分别进行说明：


- 第一步：调用OVS的add port这个动作，具体可以从`transaction()`这个方法说起。


- 第二部：查询数据库，将刚创建的port参数信息返回，具体可以从`db_set()`这个方法说起。


#### **transaction()**

neutron.agent.ovsdb.api`中，`transaction()`实际上是一个抽象的方法，具体是通过`OvsdbVsctl`类来进行实现的


```
	# 可以理解为操作命令的转换
    @abc.abstractmethod
    def transaction(self, check_error=False, log_errors=True, **kwargs):
        """Create a transaction

        :param check_error: Allow the transaction to raise an exception?
        :type check_error:  bool
        :param log_errors:  Log an error if the transaction fails?
        :type log_errors:   bool
        :returns: A new transaction
        :rtype: :class:Transaction
        """

```

```

# 继承后将实现该抽象方法
class OvsdbVsctl(ovsdb.API):
    def transaction(self, check_error=False, log_errors=True, **kwargs):
        return Transaction(self.context, check_error, log_errors, **kwargs)

```


```

# 下面来看具体实现，基类为ovsdb.Transaction
class Transaction(ovsdb.Transaction):
    def __init__(self, context, check_error=False, log_errors=True, opts=None):
        self.context = context
        self.check_error = check_error
        self.log_errors = log_errors
		# 配置log信息
        self.opts = ["--timeout=%d" % self.context.vsctl_timeout,
                     '--oneline', '--format=json']
        if opts:
            self.opts += opts
        self.commands = []

    def add(self, command):
        self.commands.append(command)
        return command

    def commit(self):
        args = []
        for cmd in self.commands:
            cmd.result = None
            args += cmd.vsctl_args()
        res = self.run_vsctl(args)
        if res is None:
            return
        res = res.replace(r'\\', '\\').splitlines()
        for i, record in enumerate(res):
            self.commands[i].result = record
        return [cmd.result for cmd in self.commands]

	# 直接执行 ovs-vsctl 操作，比如add-port,add-br等操作
    def run_vsctl(self, args):
        full_args = ["ovs-vsctl"] + self.opts + args
        try:
            # We log our own errors, so never have utils.execute do it
            return utils.execute(full_args, run_as_root=True,
                                 log_fail_as_error=False).rstrip()
        except Exception:
            with excutils.save_and_reraise_exception() as ctxt:
                if self.log_errors:
                    LOG.exception(_LE("Unable to execute %(cmd)s."),
                                  {'cmd': full_args})
                if not self.check_error:
                    ctxt.reraise = False

```

那么这就将neutron的API命令转换为OVS命令调用OVS进行网络的配置。



#### **db_set()** 

主要从数据库中读取新创建的port信息。

从`get_port_ofport()`入手分析，

```
    def get_port_ofport(self, port_name):
        """Get the port's assigned ofport, retrying if not yet assigned."""
        ofport = INVALID_OFPORT
        try:
		# 可以看到实际上是调用了内部方法_get_port_ofport来得到ofport
            ofport = self._get_port_ofport(port_name)
        except retrying.RetryError as e:
            LOG.exception(_LE("Timed out retrieving ofport on port %(pname)s. "
                              "Exception: %(exception)s"),
                          {'pname': port_name, 'exception': e})
        return ofport

```

那么_get_port_ofport又进行了什么动作呢：

```
	# 添加了@_ofport_retry装饰器，将调用属性直接转换为调用方法
    @_ofport_retry
    def _get_port_ofport(self, port_name):
	# 方法返回了数据库API的一个方法db_get_val
        return self.db_get_val("Interface", port_name, "ofport")

```

那么db_get_val是如何运行的呢：


```

    def db_get_val(self, table, record, column, check_error=False,
                   log_errors=True):
	# 直接返回了数据库的db_get方法结果，接着来看db_get的具体操作
        return self.ovsdb.db_get(table, record, column).execute(
            check_error=check_error, log_errors=log_errors)

```

可以看到实际上是调用了ovsdb的`db_get`方法
在`neutron.agent.ovsdb.api`中，`db_get`实际上是一个抽象的方法，具体是通过`OvsdbVsctl`类来进行实现的

```

# 子类来实现ovsdb.API
class OvsdbVsctl(ovsdb.API):
	# 返回dict数组的port信息
    def db_get(self, table, record, column):
        # Use the 'list' command as it can return json and 'get' cannot so that
        # we can get real return types instead of treating everything as string
        # NOTE: openvswitch can return a single atomic value for fields that
        # are sets, but only have one value. This makes directly iterating over
        # the result of a db_get() call unsafe.
        return DbGetCommand(self.context, 'list', args=[table, record],
                            columns=[column])

```

```

class DbGetCommand(DbCommand):
    @DbCommand.result.setter
    def result(self, val):
        # super()'s never worked for setters http://bugs.python.org/issue14965'
        DbCommand.result.fset(self, val)
        # DbCommand will return [{'column': value}] and we just want value.
        if self._result:
            self._result = list(self._result[0].values())[0]

```

返回一个元数组的查询结果。


> python学习小结：super()的好处就是可以避免直接使用父类的名字.但是它主要用于多重继承，在Python3.0中可以super(OVSBridge, self).__init__()替换为super().__init__()
 

```
# 相同功能的两种实现方式
class Base(object):
    def __init__(self):
        print "Base created"

class ChildA(Base):
    def __init__(self):
        Base.__init__(self)

class ChildB(Base):
    def __init__(self):
        super(ChildB, self).__init__()

print ChildA(),ChildB()

```

super[更加详细的介绍](http://www.artima.com/weblogs/viewpost.jsp?thread=236275)
