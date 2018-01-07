# NetManage

This is a tool to configure network management protocols (SNMP, Syslog, SSH, NTP) on network devices and servers. This tool can be used as a super-quick way to setup many network-attached devices for monitoring via NMS/PAM.

### Before you begin: inventory

Please fill in Ansible inventory file before you begin. The inventory file should contain the devices to be managed. 

Linux- and Python-enabed are natively supported via Ansible and device types will be learned via Ansible setup module. Other devices (like FortiOS, Exreme Networks) need ansible_os to be specified.
Please see sample inventory file below.

```
[managed_devices]
cumulus.local
extreme.local ansible_os=exos
fortigate.local ansible_os=fortios fortios_vdom="global"
```

### Before you begin: model

Please fill in management protocols params inside group_vars/managed_devices file. This data will be used to configure management protocols on network devices.

```
vrf: mgmt

snmp:
  contact: "admin@local"
  location: "datacenter"
  v3:
    - server_ip: 1.1.1.1
      user: monitor
      auth: 123456
      enc: 654321

ntp_servers:
  - 192.168.1.10
  - 192.168.1.20

syslog_servers:
  - 192.168.5.14


ssh:
  - user: monitor
    enc_pass: $6$jaSl0NPh$h9cD1D8OQbGRW99sOl0lMlLabVx5p8eCvZQAYdOqtGkSKlcr/93HRb.O2phIR9FSSI6EOJYLDXUGfVxWyXLEx/
    # 123456
    # mkpasswd --method=SHA-512 <pass>

```

### Before you begin: deployment user


To run Ansible playbooks you need a user (deployment user) to connect to network devices. You can use simple Ansible playbooks inside /utils directory to set up the deployment  user.


```
ansible-playbook -i inventory.sample -l cumulus.local cumulus-add-user.yml -e "ansible_ssh_pass=CumulusLinux!"
```

To add deployment user to network devices like Extreme XOS you can use Ansible raw module. Please note that the module will "fail" though commands are executed successfully. 


```
ansible -i inventiry.sample -l -m raw -c paramiko -a "create account admin ..."
extreme.local | FAILED | rc=-1 >>

non-zero return code

```


### ...and action!

The following playbook will configure a set of management protocols. Please note that not every <device type,protocol> pairs are implemented in playbook. Please see getters/* for details. 

```
ansible-playbook -i inventory.sample deploy-management.yml --tags="snmp,syslog,ntp,ssh"
```

You can use the playbook with clean=1 parameter to run kind of old-config cleaning action before deploying desired config.

```
ansible-playbook -i inventory.sample deploy-management.yml --tags="snmp,syslog,ntp,ssh" -e "clean=1"
```
### Logging

Ansible logging to file is enabled by default. You will be able to see latest playbook run in ./logging/latest.log file. The ./logging directory also has all previous playbook runs named after timestamp when the playbook was run.

Please note: running 2 or more playbooks at the same time very likely will lead to corrupted log file.  


### Unit testing

You can run unit tests of a templating procedure

```
ansible-playbook -i tests/dummy_inventory run-tests.yml
```

The playbook will take sample inputs (inventory group_vars files) from tests/group_vars/* and calls templating procedure. Each sample input has an expected result ('ok', or 'fail:<fail scope>'). As the result of the test you will see if sample outputs met expectations.

If tests went successfully:

```
TASK [Checking templating results [ OK (unexpected) ]] *************************

TASK [Checking templating results [ FAILED (unexpected) ]] *********************

TASK [Results] *****************************************************************
ok: [cumulus] => {
    "msg": [
        ">>>", 
        ">>> Please check the outputs above for errors if there are any:", 
        ">>>  * OK (unexpected) => templating must have failed, but is OK", 
        ">>>  * FAILED (unexpected) => templating must be OK, but failed", 
        ">>>   ", 
        ""
    ]
}

```  

If some tests didn't meet expected result:

```
TASK [Checking templating results [ OK (unexpected) ]] **********************************************

TASK [Checking templating results [ FAILED (unexpected) ]] ******************************************
failed: [cumulus] (item={'failed': True, 'model_file': u'syslog.cumulus.j2', 'test_file': u'test2.ok'}) => {
    "changed": false, 
    "failed_when_result": true, 
    "msg": [
        ">>>", 
        ">>> FAILED (unexpected)", 
        ">>>", 
        ""
    ], 
    "tr": {
        "failed": true, 
        "model_file": "syslog.cumulus.j2", 
        "test_file": "test2.ok"
    }
}

```

You can write your own tests and place them in tests/group_vars/ directory. New tests will be used on next run.

### TO DO

Undeploy playbook might be needed to remove per-protocol configuration. However you might define new configuration data and re-run deploy playbook with clean=1 flag to get old configuration deleted (re-written). 
