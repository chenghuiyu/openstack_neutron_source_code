## **dvr_router_base.py**
----------------------------------

### **DvrRouterBase类**

继承于父类`router.RouterInfo`

```
    def process(self, agent):
        super(DvrRouterBase, self).process(agent)
        # NOTE:  Keep a copy of the interfaces around for when they are removed
        self.snat_ports = self.get_snat_interfaces()
	# 从dict中根据snat_routee_inte_key来获取interface ports信息
    def get_snat_interfaces(self):
        return self.router.get(l3_constants.SNAT_ROUTER_INTF_KEY, [])

	# 带有snat功能的router有多个interface port，下面就是如何找到具有snat功能的port
    def get_snat_port_for_internal_port(self, int_port, snat_ports=None):
        """Return the SNAT port for the given internal interface port."""
        if snat_ports is None:
            snat_ports = self.get_snat_interfaces()
        fixed_ip = int_port['fixed_ips'][0]
        subnet_id = fixed_ip['subnet_id']
        if snat_ports:
		# 根据list内的元素找到指定subnet_id的port，如果满足条件直接返回
            match_port = [p for p in snat_ports
                          if p['fixed_ips'][0]['subnet_id'] == subnet_id]
            if match_port:
                return match_port[0]
            else:
                LOG.error(_LE('DVR: SNAT port not found in the list '
                              '%(snat_list)s for the given router '
                              ' internal port %(int_p)s'), {
                                  'snat_list': snat_ports,
                                  'int_p': int_port})


```
