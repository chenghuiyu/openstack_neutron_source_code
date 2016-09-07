# neutron中的QOS实现源码-api中的实现

------------------------------------------------------

这部分的QOS主要在`neutron.api.rpc.callbacks.resource.py`中实现的。


```
_QOS_POLICY_CLS = policy.QosPolicy

_VALID_CLS = (
    _QOS_POLICY_CLS,
)

_VALID_TYPES = [cls.obj_name() for cls in _VALID_CLS]


# Supported types
QOS_POLICY = _QOS_POLICY_CLS.obj_name()


_TYPE_TO_CLS_MAP = {
    QOS_POLICY: _QOS_POLICY_CLS,
}


def get_resource_type(resource_cls):
    if not resource_cls:
        return None

    if not hasattr(resource_cls, 'obj_name'):
        return None

    return resource_cls.obj_name()


def is_valid_resource_type(resource_type):
    return resource_type in _VALID_TYPES


def get_resource_cls(resource_type):
    return _TYPE_TO_CLS_MAP.get(resource_type)

```




