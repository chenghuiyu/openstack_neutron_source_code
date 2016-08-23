## **config.py**

主要定义了一些有关agent的一些配置信息的关键字和默认值，同时声明了一些注册配置的函数。

```
# root_helper_opts配置参数

ROOT_HELPER_OPTS = [
    cfg.StrOpt('root_helper', default='sudo',
               help=_('Root helper application.')),
    cfg.BoolOpt('use_helper_for_ns_read',
                default=True,
                help=_('Use the root helper to read the namespaces from '
                       'the operating system.')),
    # We can't just use root_helper=sudo neutron-rootwrap-daemon $cfg because
    # it isn't appropriate for long-lived processes spawned with create_process
    # Having a bool use_rootwrap_daemon option precludes specifying the
    # rootwrap daemon command, which may be necessary for Xen?
    cfg.StrOpt('root_helper_daemon',
               help=_('Root helper daemon application to use when possible.')),
]

# agent_state_opts配置参数
# agent的状态信息读取，包括report_interval时间间隔，心跳信息等
AGENT_STATE_OPTS = [
    cfg.FloatOpt('report_interval', default=30,
                 help=_('Seconds between nodes reporting state to server; '
                        'should be less than agent_down_time, best if it '
                        'is half or less than agent_down_time.')),
    cfg.BoolOpt('log_agent_heartbeats', default=False,
                help=_('Log agent heartbeats')),
]
# interface_driver_opts
# 接口的驱动参数
INTERFACE_DRIVER_OPTS = [
    cfg.StrOpt('interface_driver',
               help=_("The driver used to manage the virtual interface.")),
]

# use_namespaces_opts
# 用户的命名空间配置参数
USE_NAMESPACES_OPTS = [
    cfg.BoolOpt('use_namespaces', default=True,
                help=_("Allow overlapping IP. This option is deprecated and "
                       "will be removed in a future release."),
                deprecated_for_removal=True),
]
# iptables配置参数
IPTABLES_OPTS = [
    cfg.BoolOpt('comment_iptables_rules', default=True,
                help=_("Add comments to iptables rules.")),
]
# process_monitor_opts配置参数，来监控子进程的动作
PROCESS_MONITOR_OPTS = [
    cfg.StrOpt('check_child_processes_action', default='respawn',
               choices=['respawn', 'exit'],
               help=_('Action to be executed when a child process dies')),
    cfg.IntOpt('check_child_processes_interval', default=60,
               help=_('Interval between checks of child process liveness '
                      '(seconds), use 0 to disable')),
]
```

### **get_log_args()方法**

主要设置一些日志处理的参数，根据配置文件里配置的参数，分别输出相应的日志信息。

### **setup_conf()方法**

```
def setup_conf():
    bind_opts = [
        # 配置默认的安装路径
        cfg.StrOpt('state_path',
                   default='/var/lib/neutron',
                   help=_('Top-level directory for maintaining dhcp state')),
    ]
```


