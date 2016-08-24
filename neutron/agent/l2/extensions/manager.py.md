## **manager.py**
-------------------------------

从配置文件中获取配置的extensions相应的关键字是`neutron.agent.l2.extensions`


### **AgentExtensionsManager类**

继承父类stevedore.named.NamedExtensionManager，用来管理agent extensions

首先根据driver_type的类型建立connection，初始化extension的后端实现，然后实现了`handle_port()`和`delete_port()`两个方法。

```
    def handle_port(self, context, data):
        """Notify all agent extensions to handle port."""
        for extension in self:
            try:
                extension.obj.handle_port(context, data)
            # TODO(QoS) add agent extensions exception and catch them here
            except AttributeError:
                LOG.exception(
                    _LE("Agent Extension '%(name)s' failed "
                        "while handling port update"),
                    {'name': extension.name}
                )

    def delete_port(self, context, data):
        """Notify all agent extensions to delete port."""
        for extension in self:
            try:
                extension.obj.delete_port(context, data)
            # TODO(QoS) add agent extensions exception and catch them here
            # instead of AttributeError
            except AttributeError:
                LOG.exception(
                    _LE("Agent Extension '%(name)s' failed "
                        "while handling port deletion"),
                    {'name': extension.name}
                )

```



