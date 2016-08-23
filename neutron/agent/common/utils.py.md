## **utils.py**


```
# 开始也是先导入不同平台下的utils
if os.name == 'nt':
    from neutron.agent.windows import utils
else:
    from neutron.agent.linux import utils


LOG = logging.getLogger(__name__)
config.register_root_helper(cfg.CONF)


# 执行命令execute
execute = utils.execute

# 从配置文件中加载接口驱动
def load_interface_driver(conf):
    if not conf.interface_driver:
        LOG.error(_LE('An interface driver must be specified'))
        raise SystemExit(1)
    try:
        return importutils.import_object(conf.interface_driver, conf)
    except ImportError as e:
        LOG.error(_LE("Error importing interface driver "
                      "'%(driver)s': %(inner)s"),
                  {'driver': conf.interface_driver,
                   'inner': e})
        raise SystemExit(1)

```



