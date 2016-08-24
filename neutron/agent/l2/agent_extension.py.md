## **agent_extension.py**
-----------------------------

定义了agent extension的抽象接口，通过其他的类来实现抽象方法


```
    @abc.abstractmethod
    def handle_port(self, context, data):
        """Handle agent extension for port.

        This can be called on either create or update, depending on the
        code flow. Thus, it's this function's responsibility to check what
        actually changed.

        :param context - rpc context
        :param data - port data
        """

    @abc.abstractmethod
    def delete_port(self, context, data):
        """Delete port from agent extension.

        :param context - rpc context
        :param data - port data
        """

```
