# ansible-linux-monitoring
Ansible playbooks to install a monitoring stack based on node-exporter, prometheus and grafana on RHEL/CentOS.

## Define an inventory
Create an inventory file and distribute roles as required. In the easiest form:
```
$ cat hosts
[all]
example.local.lab

[node_exporter]
example.local.lab

[prometheus]
example.local.lab

[grafana]
example.local.lab
```

## Install monitoring stack
Run install-all.yaml:

```
$ ansible-playbook -i ./hosts ./install-all.yaml

PLAY [Install node-exporter container and systemd service] ***********************************************************

TASK [node-exporter : Create systemd services for node-exporter container] *******************************************
ok: [example.local.lab]

TASK [node-exporter : Reload systemd] ********************************************************************************
skipping: [example.local.lab]

TASK [node-exporter : Ensure node-exporter container is running] *****************************************************
ok: [example.local.lab]

PLAY [Install prometheus container and systemd service] **************************************************************

TASK [prometheus : Create podman volume for prometheus configuration] ************************************************
ok: [example.local.lab]

TASK [prometheus : Template prometheus.yml.j2 to host] ***************************************************************
changed: [example.local.lab]

TASK [prometheus : Create tar with prometheus.yml] *******************************************************************
changed: [example.local.lab]

TASK [prometheus : Populate prometheus volume with tar file] *********************************************************
changed: [example.local.lab]

TASK [prometheus : Remove temporary files] ***************************************************************************
changed: [example.local.lab] => (item=/tmp/prometheus.yml)
changed: [example.local.lab] => (item=/tmp/prometheus.yml.tar)

TASK [prometheus : Create systemd services for prometheus container] *************************************************
changed: [example.local.lab]

TASK [prometheus : Reload systemd] ***********************************************************************************
ok: [example.local.lab]

TASK [prometheus : Ensure prometheus container is running] ***********************************************************
changed: [example.local.lab]

PLAY [Install grafana container and systemd service] *****************************************************************

TASK [grafana : Create podman volume for grafana data] ***************************************************************
changed: [example.local.lab]

TASK [grafana : Create systemd services for grafana container] *******************************************************
changed: [example.local.lab]

TASK [grafana : Reload systemd] **************************************************************************************
ok: [example.local.lab]

TASK [grafana : Ensure grafana container is running] *****************************************************************
changed: [example.local.lab]

TASK [grafana : Wait for port 3000 to be up] *************************************************************************
ok: [example.local.lab]

TASK [grafana : Create prometheus datasource] ************************************************************************
ok: [example.local.lab] => (item=example.local.lab)

PLAY RECAP ***********************************************************************************************************
example.local.lab : ok=7   changed=10    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```

# Uninstall monitoring stack
Same as above, but run with an environment variable like this 'task=uninstall':
```
$ ansible-playbook -i ./hosts ./install-all.yaml -e 'task=uninstall'
```
