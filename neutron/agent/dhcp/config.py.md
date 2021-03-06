## **config.py**

---------------------------

主要定义了dhcp的一些关键字和配置参数，以及一些注册函数

```
# DHCP agent的配置信息
DHCP_AGENT_OPTS = [
    cfg.IntOpt('resync_interval', default=5,
               help=_("Interval to resync.")),
    cfg.StrOpt('dhcp_driver',
               default='neutron.agent.linux.dhcp.Dnsmasq',
               help=_("The driver used to manage the DHCP server.")),
    cfg.BoolOpt('enable_isolated_metadata', default=False,
                help=_("Support Metadata requests on isolated networks.")),
    cfg.BoolOpt('force_metadata', default=False,
                help=_("Force to use DHCP to get Metadata on all networks.")),
    cfg.BoolOpt('enable_metadata_network', default=False,
                help=_("Allows for serving metadata requests from a "
                       "dedicated network. Requires "
                       "enable_isolated_metadata = True")),
    cfg.IntOpt('num_sync_threads', default=4,
               help=_('Number of threads to use during sync process.'))
]
# DHCP 的配置信息
DHCP_OPTS = [
    cfg.StrOpt('dhcp_confs',
               default='$state_path/dhcp',
               help=_('Location to store DHCP server config files')),
    cfg.StrOpt('dhcp_domain',
               default='openstacklocal',
               help=_('Domain to use for building the hostnames.'
                      'This option is deprecated. It has been moved to '
                      'neutron.conf as dns_domain. It will removed from here '
                      'in a future release'),
               deprecated_for_removal=True),
]
# DNS 消息队列的配置信息
DNSMASQ_OPTS = [
    cfg.StrOpt('dnsmasq_config_file',
               default='',
               help=_('Override the default dnsmasq settings with this file')),
    cfg.ListOpt('dnsmasq_dns_servers',
                help=_('Comma-separated list of the DNS servers which will be '
                       'used as forwarders.'),
                deprecated_name='dnsmasq_dns_server'),
    cfg.BoolOpt('dhcp_delete_namespaces', default=True,
                help=_("Delete namespace after removing a dhcp server."
                       "This option is deprecated and "
                       "will be removed in a future release."),
                deprecated_for_removal=True),
    cfg.StrOpt('dnsmasq_base_log_dir',
               help=_("Base log dir for dnsmasq logging. "
                      "The log contains DHCP and DNS log information and "
                      "is useful for debugging issues with either DHCP or "
                      "DNS. If this section is null, disable dnsmasq log.")),
    cfg.IntOpt(
        'dnsmasq_lease_max',
        default=(2 ** 24),
        help=_('Limit number of leases to prevent a denial-of-service.')),
    cfg.BoolOpt('dhcp_broadcast_reply', default=False,
                help=_("Use broadcast in DHCP replies")),
]

```




