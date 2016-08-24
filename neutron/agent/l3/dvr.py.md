## **dvr.py**

------------------------------------------

### **AgentMixin类**

这个类实现的什么功能呢？





floating ip的namespace，即变量`_fip_namespaces`采用弱引用，方便垃圾回收

```
self._fip_namespaces = weakref.WeakValueDictionary()

```

#### **get_fip_ns()**

主要用来获取floating IP的namespace，

```
    def get_fip_ns(self, ext_net_id):
        # TODO(Carl) is this necessary?  Code that this replaced was careful to
        # convert these to string like this so I preserved that.
		# 这里应该是外部网络分配的floating IP的ID，相当于key来进行查询
        ext_net_id = str(ext_net_id)

		# 弱引用中的get方法，根据ext_net_id获取到fip_ns
        fip_ns = self._fip_namespaces.get(ext_net_id)
		# 如果不是需要回收的就直接返回fip_ns
        if fip_ns and not fip_ns.destroyed:
            return fip_ns

		# 如果当做弱引用已经被垃圾回收了，那么就根据ext_net_id和配置文件的信息重新获得fip_ns，并返回
		# FipNamespace类是namespaces.Namespace的子类
        fip_ns = dvr_fip_ns.FipNamespace(ext_net_id,
                                         self.conf,
                                         self.driver,
                                         self.use_ipv6)
        self._fip_namespaces[ext_net_id] = fip_ns

        return fip_ns

```

#### **get_ports_by_subnet()**

相当于client端的RPC调用，即L3PluginApi的get_ports_by_subnet()来根据subnet_id得到端口信息。


```
    def get_ports_by_subnet(self, subnet_id):
        return self.plugin_rpc.get_ports_by_subnet(self.context, subnet_id)

```


#### **add_arp_entry()**

主要是根据subnet在route的ns中添加ARP实体

```

    def add_arp_entry(self, context, payload):
        """Add arp entry into router namespace.  Called from RPC."""
        router_id = payload['router_id']
        ri = self.router_info.get(router_id)
        if not ri:
            return

        arp_table = payload['arp_table']
        ip = arp_table['ip_address']
        mac = arp_table['mac_address']
        subnet_id = arp_table['subnet_id']
        ri._update_arp_entry(ip, mac, subnet_id, 'add')

```
#### **del_arp_entry()**

主要是根据subnet在route的ns中删除ARP实体

```

    def del_arp_entry(self, context, payload):
        """Delete arp entry from router namespace.  Called from RPC."""
        router_id = payload['router_id']
        ri = self.router_info.get(router_id)
        if not ri:
            return

        arp_table = payload['arp_table']
        ip = arp_table['ip_address']
        mac = arp_table['mac_address']
        subnet_id = arp_table['subnet_id']
        ri._update_arp_entry(ip, mac, subnet_id, 'delete')

```

`_update_arp_entry`方法可以根据不同的操作，来进行添加和删除，这个方法位于`DvrLocalRouter`类中


```
def _update_arp_entry(self, ip, mac, subnet_id, operation):
        """Add or delete arp entry into router namespace for the subnet."""
		# 得到port信息
        port = self._get_internal_port(subnet_id)
        # update arp entry only if the subnet is attached to the router
        if not port:
            return False

        try:
            # TODO(mrsmith): optimize the calls below for bulk calls
			# 根据port信息得到device，并利用Linux包中的方法对device进行操作
            interface_name = self.get_internal_device_name(port['id'])
            device = ip_lib.IPDevice(interface_name, namespace=self.ns_name)
            if ip_lib.device_exists(interface_name, self.ns_name):
                if operation == 'add':
                    device.neigh.add(ip, mac)
                elif operation == 'delete':
                    device.neigh.delete(ip, mac)
                return True
            else:
                if operation == 'add':
                    LOG.warn(_LW("Device %s does not exist so ARP entry "
                                 "cannot be updated, will cache information "
                                 "to be applied later when the device exists"),
                             device)
                    self._cache_arp_entry(ip, mac, subnet_id, operation)
                return False
        except Exception:
            with excutils.save_and_reraise_exception():
                LOG.exception(_LE("DVR: Failed updating arp entry"))
```




> python学习小结：python中的weakref(弱引用)  使用weakref模块的WeakKeyDictionary和WeakValueDictionary类对对象进行弱引用，则可以减少资源占用，方便进行内存回收。
