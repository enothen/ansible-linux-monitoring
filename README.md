# ansible-linux-monitoring
Ansible playbooks to install a monitoring stack based on node-exporter, prometheus and grafana.

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

A more complex form of inventory, but also more realistic, would deploy node-exporter in all hosts, Prometheus in one host per site, maybe grouping by site, and finally Grafana in a single place but with data sources configured for each Prometheus instance.
The following example considers hosts on two different sites. All the hosts will get node-exporter installed. Only one host on each site will also get Prometheus, and a single (different) host will be running Grafana.
```
$ cat hosts-multisite.yaml
sitea:
  hosts:
    host1.sitea.example.lab:
    host2.sitea.example.lab:
    host3.sitea.example.lab:

siteb:
  hosts:
    host4.siteb.example.lab:
    host5.siteb.example.lab:
    host6.siteb.example.lab:
    host7.siteb.example.lab:

node_exporter:
  children:
    sitea:
    siteb:

prometheus:
  hosts:
    host1.sitea.example.lab:
      clients:
        - host1.sitea.example.lab
        - host2.sitea.example.lab
        - host3.sitea.example.lab
    host4.siteb.example.lab:
      clients:
        - host4.siteb.example.lab
        - host5.siteb.example.lab
        - host6.siteb.example.lab
        - host7.siteb.example.lab

grafana:
  hosts:
    host7.siteb.example.lab:
```

Each host running Prometheus will have, by default, all the hosts in the node_exporter group configured as targets. To override this, the example above uses the clients variable on each Prometheus host, which results in the following configuration applied on host1:
```
$ cat prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: node-exporter
    static_configs:
      - targets:
        - host1.sitea.example.lab:9100
        - host2.sitea.example.lab:9100
        - host3.sitea.example.lab:9100
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.+):(\d+)'
        target_label: instance
        replacement: '${1}'
```

The server running Grafana will have a datasource configured for each host in the prometheus group:
```
$ curl -s http://admin:admin@host7.siteb.example.lab:3000/api/datasources | jq -j '.[]|(.name, ": ", .url, "\n")'
host1.sitea.example.lab: http://host1.sitea.example.lab:9090
host4.siteb.example.lab: http://host4.siteb.example.lab:9090
```

## Install pre-requisites
Podman is the only pre-requisite on the target hosts. Either make sure it is installed in advance, or install it using the playbook provided:

```
$ ansible-playbook -i ./hosts ./install-podman.yaml

PLAY [Simple playbook to install podman on the inventory hosts] ******************************************************

TASK [Gathering Facts] ***********************************************************************************************
ok: [example.local.lab]

TASK [Install podman in Debian distros] ******************************************************************************
skipping: [example.local.lab]

TASK [Install podman in Red Hat distros] *****************************************************************************
ok: [example.local.lab]

PLAY RECAP ***********************************************************************************************************
example.local.lab : ok=2    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
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
