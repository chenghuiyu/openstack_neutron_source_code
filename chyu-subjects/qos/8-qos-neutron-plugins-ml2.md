# neutron中的QOS实现源码-ml2中的实现

------------------------------------------------------------

ml2中实现qos主要有三个文件，managers.py、plugin.py和rpc.py三个文件。

## **managers.py**

在`managers.py`中，`MechanismManager类`主要实现了qos功能，`supported_qos_rule_types()`方法来让drivers决定实现相应的qos rule

```
    def supported_qos_rule_types(self):
        if not self.ordered_mech_drivers:
            return []

        rule_types = set(qos_consts.VALID_RULE_TYPES)
        binding_driver_found = False

        # Recalculate on every call to allow drivers determine supported rule
        # types dynamically
        for driver in self.ordered_mech_drivers:
            driver_obj = driver.obj
            if driver_obj._supports_port_binding:
                binding_driver_found = True
                if hasattr(driver_obj, 'supported_qos_rule_types'):
                    new_rule_types = \
                        rule_types & set(driver_obj.supported_qos_rule_types)
                    dropped_rule_types = new_rule_types - rule_types
                    if dropped_rule_types:
                        LOG.info(
                            _LI("%(rule_types)s rule types disabled for ml2 "
                                "because %(driver)s does not support them"),
                            {'rule_types': ', '.join(dropped_rule_types),
                             'driver': driver.name})
                    rule_types = new_rule_types
                else:
                    # at least one of drivers does not support QoS, meaning
                    # there are no rule types supported by all of them
                    LOG.warn(
                        _LW("%s does not support QoS; "
                            "no rule types available"),
                        driver.name)
                    return []

        if binding_driver_found:
            rule_types = list(rule_types)
        else:
            rule_types = []
        LOG.debug("Supported QoS rule types "
                  "(common subset for all mech drivers): %s", rule_types)
        return rule_types

```

## **plugin.py**


主要实现了`Ml2Plugin`类，该类继承了多个不同的父类，
