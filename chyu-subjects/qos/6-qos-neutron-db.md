# neutron中的QOS实现源码-**db中的实现**

----------------------------------------------------


主要位于`neutron.db.qos`下面，包括`api.py`和`models.py`

## **api.py**


主要对network和port进行操作。



`oslo.db`中的`context.session`来对`sqlalchemy`进行操作，`models.QosNetworkPolicyBinding`返回数据库的操作对象（根据传入的policy_id和network_id来进行操作）

`create_policy_network_binding`对RESTful API的delete进行响应，在数据库中创建和network绑定的policy策略 

```
def create_policy_network_binding(context, policy_id, network_id):
    try:
        with context.session.begin(subtransactions=True):
            db_obj = models.QosNetworkPolicyBinding(policy_id=policy_id,
                                                    network_id=network_id)
            context.session.add(db_obj)
    except oslo_db_exception.DBReferenceError:
        raise n_exc.NetworkQosBindingNotFound(net_id=network_id,
                                              policy_id=policy_id)

```

`delete_policy_network_binding`对RESTful API的delete进行响应，在数据库中删除和network绑定的policy策略

```
def delete_policy_network_binding(context, policy_id, network_id):
    try:
        with context.session.begin(subtransactions=True):
            db_object = (db.model_query(context,
                                        models.QosNetworkPolicyBinding)
                         .filter_by(policy_id=policy_id,
                                    network_id=network_id).one())
            context.session.delete(db_object)
    except orm_exc.NoResultFound:
        raise n_exc.NetworkQosBindingNotFound(net_id=network_id,
                                              policy_id=policy_id)

```

`create_policy_port_binding`对RESTful API的create进行响应，在数据库中创建和port绑定的policy策略

```
def create_policy_port_binding(context, policy_id, port_id):
    try:
        with context.session.begin(subtransactions=True):
            db_obj = models.QosPortPolicyBinding(policy_id=policy_id,
                                                 port_id=port_id)
            context.session.add(db_obj)
    except oslo_db_exception.DBReferenceError:
        raise n_exc.PortQosBindingNotFound(port_id=port_id,
                                           policy_id=policy_id)

```


`delete_policy_port_binding`对RESTful API的delete进行响应，在数据库中删除和port绑定的policy策略

```
def delete_policy_port_binding(context, policy_id, port_id):
    try:
        with context.session.begin(subtransactions=True):
            db_object = (db.model_query(context, models.QosPortPolicyBinding)
                         .filter_by(policy_id=policy_id,
                                    port_id=port_id).one())
            context.session.delete(db_object)
    except orm_exc.NoResultFound:
        raise n_exc.PortQosBindingNotFound(port_id=port_id,
                                           policy_id=policy_id)

```


`session`是通过sqlalchemy来操作数据库的入口，session的功能主要就是管理程序和数据库之间的会话，对于CRUD的增删改查操作就是通过session对象来进行操作的。上述的四个方法：`create_policy_network_binding`、`delete_policy_network_binding`、`create_policy_port_binding`和`delete_policy_port_binding`就是通过session实现的。



## **modles.py**


`QosPolicy`继承父类`model_base.BASEV2`、`models_v2.HasId`、`models_v2.HasTenant`，在neutron的对象关系映射处理中，使用SQLAlchemy数据库进行ORM实现，所以在`models.py`中，都采用SQLAlchemy进行管理以及操作。在对类中的一个对象进行操作的时候，直接映射为数据库中的一张表，方便了增删改查。这里总共维护了四张表：`QosPolicy`，`QosNetworkPolicyBinding`，`QosPortPolicyBinding`，`QosBandwidthLimitRule`

```
class QosPolicy(model_base.BASEV2, models_v2.HasId, models_v2.HasTenant):
    __tablename__ = 'qos_policies'
    name = sa.Column(sa.String(attrs.NAME_MAX_LEN))
    description = sa.Column(sa.String(attrs.DESCRIPTION_MAX_LEN))
    shared = sa.Column(sa.Boolean, nullable=False)

```

首先定义一个映射类`QosNetworkPolicyBinding`，后续可以通过操作这个类的实例来实现对数据库表`QosNetworkPolicyBinding`的操作，在映射类中，通过使用`__tablename__`属性来指定该映射类所对应的数据库表，通过`Column`类实例的方式来指定数据库的字段。



```
class QosNetworkPolicyBinding(model_base.BASEV2):
    __tablename__ = 'qos_network_policy_bindings'
    policy_id = sa.Column(sa.String(36),
                          sa.ForeignKey('qos_policies.id',
                                        ondelete='CASCADE'),
                          nullable=False,
                          primary_key=True)
    network_id = sa.Column(sa.String(36),
                           sa.ForeignKey('networks.id',
                                         ondelete='CASCADE'),
                           nullable=False,
                           unique=True,
                           primary_key=True)
    network = sa.orm.relationship(
        models_v2.Network,
        backref=sa.orm.backref("qos_policy_binding", uselist=False,
                               cascade='delete', lazy='joined'))

```



```
class QosPortPolicyBinding(model_base.BASEV2):
    __tablename__ = 'qos_port_policy_bindings'
    policy_id = sa.Column(sa.String(36),
                          sa.ForeignKey('qos_policies.id',
                                        ondelete='CASCADE'),
                          nullable=False,
                          primary_key=True)
    port_id = sa.Column(sa.String(36),
                        sa.ForeignKey('ports.id',
                                      ondelete='CASCADE'),
                        nullable=False,
                        unique=True,
                        primary_key=True)
    port = sa.orm.relationship(
        models_v2.Port,
        backref=sa.orm.backref("qos_policy_binding", uselist=False,
                               cascade='delete', lazy='joined'))

```



```
class QosBandwidthLimitRule(models_v2.HasId, model_base.BASEV2):
    __tablename__ = 'qos_bandwidth_limit_rules'
    qos_policy_id = sa.Column(sa.String(36),
                              sa.ForeignKey('qos_policies.id',
                                            ondelete='CASCADE'),
                              nullable=False,
                              unique=True)
    max_kbps = sa.Column(sa.Integer)
    max_burst_kbps = sa.Column(sa.Integer)

```



