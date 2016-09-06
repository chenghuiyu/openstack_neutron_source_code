# neutron中的QOS实现源码-**common中的实现**

------------------------------------------


qos在这一部分中主要是出现异常后的处理方法

## `NeutronException基类`

该基类是基本的neutron的异常类，如果在qos出现异常时，就直接继承该类，并抛出异常。


```
class NeutronException(Exception):
    """Base Neutron Exception.

    To correctly use this class, inherit from it and define
    a 'message' property. That message will get printf'd
    with the keyword arguments provided to the constructor.
    """
    message = _("An unknown exception occurred.")

    def __init__(self, **kwargs):
        try:
            super(NeutronException, self).__init__(self.message % kwargs)
            self.msg = self.message % kwargs
        except Exception:
            with excutils.save_and_reraise_exception() as ctxt:
                if not self.use_fatal_exceptions():
                    ctxt.reraise = False
                    # at least get the core message out if something happened
                    super(NeutronException, self).__init__(self.message)

    if six.PY2:
        def __unicode__(self):
            return unicode(self.msg)

    def __str__(self):
        return self.msg

    def use_fatal_exceptions(self):
        return False

```



`NotFound类`继承自NeutronException基类，如果在neutron中出现not found的情况，那么就可以继承`NotFound类`直接抛出异常。

```
class NotFound(NeutronException):
    pass

class InUse(NeutronException):
    message = _("The resource is inuse")


```



这里的有关qos的异常类，主要继承自`NotFound`和`InUse`，逻辑比较清晰，不再赘述。



```
class QosPolicyNotFound(NotFound):
    message = _("QoS policy %(policy_id)s could not be found")


class QosRuleNotFound(NotFound):
    message = _("QoS rule %(rule_id)s for policy %(policy_id)s "
                "could not be found")


class PortNotFoundOnNetwork(NotFound):
    message = _("Port %(port_id)s could not be found "
                "on network %(net_id)s")


class PortQosBindingNotFound(NotFound):
    message = _("QoS binding for port %(port_id)s and policy %(policy_id)s "
                "could not be found")


class NetworkQosBindingNotFound(NotFound):
    message = _("QoS binding for network %(net_id)s and policy %(policy_id)s "
                "could not be found")


class PolicyFileNotFound(NotFound):
    message = _("Policy configuration policy.json could not be found")


class PolicyInitError(NeutronException):
    message = _("Failed to init policy %(policy)s because %(reason)s")


class PolicyCheckError(NeutronException):
    message = _("Failed to check policy %(policy)s because %(reason)s")

```



```
class QosPolicyInUse(InUse):
    message = _("QoS Policy %(policy_id)s is used by "
                "%(object_type)s %(object_id)s.")

```
