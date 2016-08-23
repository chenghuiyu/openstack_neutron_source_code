## **polling.py**
-------------------------------------

```
import os


if os.name == 'nt':
    from neutron.agent.windows import polling
else:
    from neutron.agent.linux import polling

get_polling_manager = polling.get_polling_manager

```

该类放在neutron.agent.linux中进行详细的说明
