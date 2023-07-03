# Apply OCP4 node configuration via MachineConfig using a Canary Rollout approach

All the following stepts are tested in OpenShift 4.13.1

## Initial idea

I need to perform a config change in my OCP nodes, but I'm not really sure that this modifications will work as I expect.<br>
If I could test it only in one or two nodes I would be a happy Platform Maintainer.

## Initial Status

All the compute nodes have the default chrony configuration, but now I want to use another NTP Servers.

## Step by Step

- New MachiConfigPool (MCP), where we are going to moving the nodes after the config validation
- MachineConfig (MC) resource related to the previous created MCP
- Relabeling of one node to receive the new configurations
- Verify that the config was correctly applied
- Move all the rest of nodes to the MCP
- Apply the MC to the "old" MCP (where the nodes comes previously)
- Delete the additional label in the nodes for them to come back to the original MCP

The MachineConfigOperator will detect than the applied configuration and the rendered one in the "old" MCP it's the same, and the nodes are not going to be rebooted.

# Test time

## Initial Cluster status

```bash
$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.13.1    True        False         18h     Cluster version is 4.13.1


$ oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-f7e23a7d6b8d6c47a48f6fb4e0080f06   True      False      False      3              3                   3                     0                      18h
worker   rendered-worker-e7736477cda652d372684230110b8fa2   True      False      False      3              3                   3                     0                      18h


$ oc get mc
NAME                                               GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
00-master                                          1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             18h
00-worker                                          1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             18h
01-master-container-runtime                        1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             18h
01-master-kubelet                                  1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             18h
01-worker-container-runtime                        1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             18h
01-worker-kubelet                                  1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             18h
99-master-generated-registries                     1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             18h
99-master-ssh                                                                                 3.2.0             18h
99-worker-generated-registries                     1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             18h
99-worker-ssh                                                                                 3.2.0             18h
rendered-master-85d1c14b374db56103ef04a6985b3638   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             18h
rendered-master-f7e23a7d6b8d6c47a48f6fb4e0080f06   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             18h
rendered-worker-282067975d53d79608eab4edffc6ddbe   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             18h
rendered-worker-e7736477cda652d372684230110b8fa2   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             18h


$ oc get no
NAME                                         STATUS   ROLES                  AGE   VERSION
ip-10-0-130-100.eu-west-1.compute.internal   Ready    control-plane,master   18h   v1.26.3+b404935
ip-10-0-154-21.eu-west-1.compute.internal    Ready    worker                 18h   v1.26.3+b404935
ip-10-0-176-246.eu-west-1.compute.internal   Ready    worker                 18h   v1.26.3+b404935
ip-10-0-179-65.eu-west-1.compute.internal    Ready    control-plane,master   18h   v1.26.3+b404935
ip-10-0-198-130.eu-west-1.compute.internal   Ready    worker                 18h   v1.26.3+b404935
ip-10-0-216-166.eu-west-1.compute.internal   Ready    control-plane,master   18h   v1.26.3+b404935
```

## Creating the temporary MachineConfigPool

TL;DR;
```bash
oc apply -f yaml/machineconfigpool.yaml
```

The yaml content is the following
```yaml
---
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: worker-config-test
spec:
  machineConfigSelector:
    matchExpressions:
    - key: machineconfiguration.openshift.io/role
      operator: In
      values:
      - worker
      - testing
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/testing: ""
  paused: false
```

The cluster status this resource creation

```bash
$ oc get mc
oNAME                                                           GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
00-master                                                      1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
00-worker                                                      1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-master-container-runtime                                    1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-master-kubelet                                              1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-worker-container-runtime                                    1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-worker-kubelet                                              1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
99-master-generated-registries                                 1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
99-master-ssh                                                                                             3.2.0             19h
99-worker-generated-registries                                 1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
99-worker-ssh                                                                                             3.2.0             19h
rendered-master-154d86fafe809ecf601542f6d428c0ee               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             34m
rendered-master-85d1c14b374db56103ef04a6985b3638               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
rendered-master-abd56ebc6830245cbef4ecff1aaf6e15               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             31m
rendered-master-f7e23a7d6b8d6c47a48f6fb4e0080f06               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
rendered-worker-12a3611d04fb7bb004569b75c3ae9bc0               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             31m
rendered-worker-282067975d53d79608eab4edffc6ddbe               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
rendered-worker-472b63d4d46ed3a2b648633cfd32a96c               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             34m
rendered-worker-config-test-12a3611d04fb7bb004569b75c3ae9bc0   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             84s
rendered-worker-e7736477cda652d372684230110b8fa2               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h


$ oc get mcp
NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-abd56ebc6830245cbef4ecff1aaf6e15               True      False      False      3              3                   3                     0                      19h
worker               rendered-worker-12a3611d04fb7bb004569b75c3ae9bc0               True      False      False      3              3                   3                     0                      19h
worker-config-test   rendered-worker-config-test-12a3611d04fb7bb004569b75c3ae9bc0   True      False      False      0              0                   0                     0                      91s
```

## Labeling one worker node to move it to the new MachineConfigPool

```bash
oc label node <node name> node-role.kubernetes.io/testing=
```

```bash
$ oc label node ip-10-0-154-21.eu-west-1.compute.internal node-role.kubernetes.io/testing=
node/ip-10-0-154-21.eu-west-1.compute.internal labeled


$ oc get no
NAME                                         STATUS   ROLES                  AGE   VERSION
ip-10-0-130-100.eu-west-1.compute.internal   Ready    control-plane,master   19h   v1.26.3+b404935
ip-10-0-154-21.eu-west-1.compute.internal    Ready    testing,worker         19h   v1.26.3+b404935
ip-10-0-176-246.eu-west-1.compute.internal   Ready    worker                 19h   v1.26.3+b404935
ip-10-0-179-65.eu-west-1.compute.internal    Ready    control-plane,master   19h   v1.26.3+b404935
ip-10-0-198-130.eu-west-1.compute.internal   Ready    worker                 19h   v1.26.3+b404935
ip-10-0-216-166.eu-west-1.compute.internal   Ready    control-plane,master   19h   v1.26.3+b404935


$ oc get mcp
NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-abd56ebc6830245cbef4ecff1aaf6e15               True      False      False      3              3                   3                     0                      19h
worker               rendered-worker-12a3611d04fb7bb004569b75c3ae9bc0               True      False      False      2              2                   2                     0                      19h
worker-config-test   rendered-worker-config-test-12a3611d04fb7bb004569b75c3ae9bc0   False     True       False      1              0                   0                     0                      2m16s


$ oc get mcp
NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-abd56ebc6830245cbef4ecff1aaf6e15               True      False      False      3              3                   3                     0                      19h
worker               rendered-worker-12a3611d04fb7bb004569b75c3ae9bc0               True      False      False      2              2                   2                     0                      19h
worker-config-test   rendered-worker-config-test-12a3611d04fb7bb004569b75c3ae9bc0   True      False      False      1              1                   1                     0                      2m37s
```

## Applying the NTP config

Using butane to generate the yamls to be applied.

The butane files content is:

```yaml
variant: openshift
version: 4.13.0
metadata:
  name: 99-testingworker-chrony 
  labels:
    machineconfiguration.openshift.io/role: testing
storage:
  files:
  - path: /etc/chrony.conf
    mode: 0644 
    overwrite: true
    contents:
      inline: |
        pool time.google.com iburst 
        driftfile /var/lib/chrony/drift
        makestep 1.0 3
        rtcsync
        logdir /var/log/chrony
```

```bash
butane butane/testing-worker-ntp.bu -o yaml/testing-worker-ntp.yaml
```

Applying the MachineConfig ressources

```bash
oc apply -f yaml/testing-worker-ntp.yaml
```

```bash
$ oc get mc
NAME                                                           GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
00-master                                                      1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
00-worker                                                      1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-master-container-runtime                                    1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-master-kubelet                                              1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-worker-container-runtime                                    1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-worker-kubelet                                              1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
99-master-generated-registries                                 1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
99-master-ssh                                                                                             3.2.0             19h
99-testingworker-chrony                                                                                   3.2.0             5s
99-worker-generated-registries                                 1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
99-worker-ssh                                                                                             3.2.0             19h
rendered-master-154d86fafe809ecf601542f6d428c0ee               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             39m
rendered-master-85d1c14b374db56103ef04a6985b3638               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
rendered-master-abd56ebc6830245cbef4ecff1aaf6e15               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             37m
rendered-master-f7e23a7d6b8d6c47a48f6fb4e0080f06               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
rendered-worker-12a3611d04fb7bb004569b75c3ae9bc0               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             37m
rendered-worker-282067975d53d79608eab4edffc6ddbe               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
rendered-worker-472b63d4d46ed3a2b648633cfd32a96c               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             39m
rendered-worker-config-test-12a3611d04fb7bb004569b75c3ae9bc0   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             6m55s
rendered-worker-e7736477cda652d372684230110b8fa2               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h


$ oc get mcp
NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-abd56ebc6830245cbef4ecff1aaf6e15               True      False      False      3              3                   3                     0                      19h
worker               rendered-worker-12a3611d04fb7bb004569b75c3ae9bc0               True      False      False      2              2                   2                     0                      19h
worker-config-test   rendered-worker-config-test-12a3611d04fb7bb004569b75c3ae9bc0   False     True       False      1              0                   0                     0                      7m6s


$ oc get no
NAME                                         STATUS                     ROLES                  AGE   VERSION
ip-10-0-130-100.eu-west-1.compute.internal   Ready                      control-plane,master   19h   v1.26.3+b404935
ip-10-0-154-21.eu-west-1.compute.internal    Ready,SchedulingDisabled   testing,worker         19h   v1.26.3+b404935
ip-10-0-176-246.eu-west-1.compute.internal   Ready                      worker                 19h   v1.26.3+b404935
ip-10-0-179-65.eu-west-1.compute.internal    Ready                      control-plane,master   19h   v1.26.3+b404935
ip-10-0-198-130.eu-west-1.compute.internal   Ready                      worker                 19h   v1.26.3+b404935
ip-10-0-216-166.eu-west-1.compute.internal   Ready                      control-plane,master   19h   v1.26.3+b404935


$ oc get no
NAME                                         STATUS   ROLES                  AGE   VERSION
ip-10-0-130-100.eu-west-1.compute.internal   Ready    control-plane,master   19h   v1.26.3+b404935
ip-10-0-154-21.eu-west-1.compute.internal    Ready    testing,worker         19h   v1.26.3+b404935
ip-10-0-176-246.eu-west-1.compute.internal   Ready    worker                 19h   v1.26.3+b404935
ip-10-0-179-65.eu-west-1.compute.internal    Ready    control-plane,master   19h   v1.26.3+b404935
ip-10-0-198-130.eu-west-1.compute.internal   Ready    worker                 19h   v1.26.3+b404935
ip-10-0-216-166.eu-west-1.compute.internal   Ready    control-plane,master   19h   v1.26.3+b404935
```

Checking the applied config

```bash
oc debug no/ip-10-0-154-21.eu-west-1.compute.internal -- bash -c 'chroot /host cat /etc/chrony.conf'

Starting pod/ip-10-0-154-21eu-west-1computeinternal-debug ...
To use host binaries, run `chroot /host`

pool time.google.com iburst 
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony

Removing debug pod ...
```

## Moving all the rest of the nodes to the MCP

```bash
$ oc get no
NAME                                         STATUS   ROLES                  AGE   VERSION
ip-10-0-130-100.eu-west-1.compute.internal   Ready    control-plane,master   19h   v1.26.3+b404935
ip-10-0-154-21.eu-west-1.compute.internal    Ready    testing,worker         18h   v1.26.3+b404935
ip-10-0-176-246.eu-west-1.compute.internal   Ready    worker                 18h   v1.26.3+b404935
ip-10-0-179-65.eu-west-1.compute.internal    Ready    control-plane,master   19h   v1.26.3+b404935
ip-10-0-198-130.eu-west-1.compute.internal   Ready    worker                 18h   v1.26.3+b404935
ip-10-0-216-166.eu-west-1.compute.internal   Ready    control-plane,master   19h   v1.26.3+b404935

oc label node ip-10-0-176-246.eu-west-1.compute.internal node-role.kubernetes.io/testing=
oc label node ip-10-0-198-130.eu-west-1.compute.internal node-role.kubernetes.io/testing=

$ oc get no
NAME                                         STATUS   ROLES                  AGE   VERSION
ip-10-0-130-100.eu-west-1.compute.internal   Ready    control-plane,master   19h   v1.26.3+b404935
ip-10-0-154-21.eu-west-1.compute.internal    Ready    testing,worker         18h   v1.26.3+b404935
ip-10-0-176-246.eu-west-1.compute.internal   Ready    testing,worker         18h   v1.26.3+b404935
ip-10-0-179-65.eu-west-1.compute.internal    Ready    control-plane,master   19h   v1.26.3+b404935
ip-10-0-198-130.eu-west-1.compute.internal   Ready    testing,worker         18h   v1.26.3+b404935
ip-10-0-216-166.eu-west-1.compute.internal   Ready    control-plane,master   19h   v1.26.3+b404935


$ oc get mcp
NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-abd56ebc6830245cbef4ecff1aaf6e15               True      False      False      3              3                   3                     0                      19h
worker               rendered-worker-12a3611d04fb7bb004569b75c3ae9bc0               True      False      False      0              0                   0                     0                      19h
worker-config-test   rendered-worker-config-test-caf389063560671ffaea78c3a35e05f7   False     True       False      3              1                   1                     0                      11m


$ oc get no
NAME                                         STATUS                     ROLES                  AGE   VERSION
ip-10-0-130-100.eu-west-1.compute.internal   Ready                      control-plane,master   19h   v1.26.3+b404935
ip-10-0-154-21.eu-west-1.compute.internal    Ready                      testing,worker         19h   v1.26.3+b404935
ip-10-0-176-246.eu-west-1.compute.internal   Ready,SchedulingDisabled   testing,worker         19h   v1.26.3+b404935
ip-10-0-179-65.eu-west-1.compute.internal    Ready                      control-plane,master   19h   v1.26.3+b404935
ip-10-0-198-130.eu-west-1.compute.internal   Ready                      testing,worker         19h   v1.26.3+b404935
ip-10-0-216-166.eu-west-1.compute.internal   Ready                      control-plane,master   19h   v1.26.3+b404935


$ oc get mcp
NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-abd56ebc6830245cbef4ecff1aaf6e15               True      False      False      3              3                   3                     0                      19h
worker               rendered-worker-12a3611d04fb7bb004569b75c3ae9bc0               True      False      False      0              0                   0                     0                      19h
worker-config-test   rendered-worker-config-test-caf389063560671ffaea78c3a35e05f7   True      False      False      3              3                   3                     0                      19m


$ oc get no
NAME                                         STATUS   ROLES                  AGE   VERSION
ip-10-0-130-100.eu-west-1.compute.internal   Ready    control-plane,master   19h   v1.26.3+b404935
ip-10-0-154-21.eu-west-1.compute.internal    Ready    testing,worker         19h   v1.26.3+b404935
ip-10-0-176-246.eu-west-1.compute.internal   Ready    testing,worker         19h   v1.26.3+b404935
ip-10-0-179-65.eu-west-1.compute.internal    Ready    control-plane,master   19h   v1.26.3+b404935
ip-10-0-198-130.eu-west-1.compute.internal   Ready    testing,worker         19h   v1.26.3+b404935
ip-10-0-216-166.eu-west-1.compute.internal   Ready    control-plane,master   19h   v1.26.3+b404935
```

## Applying the same config to the actual empty worker machine config pool

New Butane Files for the node pool

```yaml
variant: openshift
version: 4.13.0
metadata:
  name: 99-worker-chrony 
  labels:
    machineconfiguration.openshift.io/role: worker
storage:
  files:
  - path: /etc/chrony.conf
    mode: 0644 
    overwrite: true
    contents:
      inline: |
        pool time.google.com iburst 
        driftfile /var/lib/chrony/drift
        makestep 1.0 3
        rtcsync
        logdir /var/log/chrony
```

```bash
butane butane/worker-ntp.bu -o yaml/worker-ntp.yaml

oc apply -f yaml/worker-ntp.yaml
```

```bash
$ oc get mc
NAME                                                           GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
00-master                                                      1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
00-worker                                                      1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-master-container-runtime                                    1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-master-kubelet                                              1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-worker-container-runtime                                    1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-worker-kubelet                                              1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
99-master-generated-registries                                 1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
99-master-ssh                                                                                             3.2.0             19h
99-testingworker-chrony                                                                                   3.2.0             14m
99-worker-chrony                                                                                          3.2.0             26s
99-worker-generated-registries                                 1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
99-worker-ssh                                                                                             3.2.0             19h
rendered-master-154d86fafe809ecf601542f6d428c0ee               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             54m
rendered-master-85d1c14b374db56103ef04a6985b3638               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
rendered-master-abd56ebc6830245cbef4ecff1aaf6e15               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             51m
rendered-master-f7e23a7d6b8d6c47a48f6fb4e0080f06               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
rendered-worker-12a3611d04fb7bb004569b75c3ae9bc0               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             51m
rendered-worker-282067975d53d79608eab4edffc6ddbe               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
rendered-worker-472b63d4d46ed3a2b648633cfd32a96c               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             54m
rendered-worker-caf389063560671ffaea78c3a35e05f7               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             21s
rendered-worker-config-test-12a3611d04fb7bb004569b75c3ae9bc0   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             21m
rendered-worker-config-test-caf389063560671ffaea78c3a35e05f7   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             14m
rendered-worker-e7736477cda652d372684230110b8fa2               1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
```

## All the nodes to the original node pool
Removing the node label

```bash
oc label node ip-10-0-154-21.eu-west-1.compute.internal node-role.kubernetes.io/testing-
oc label node ip-10-0-176-246.eu-west-1.compute.internal node-role.kubernetes.io/testing-
oc label node ip-10-0-198-130.eu-west-1.compute.internal node-role.kubernetes.io/testing-
```

```bash
$ oc get no
NAME                                         STATUS   ROLES                  AGE   VERSION
ip-10-0-130-100.eu-west-1.compute.internal   Ready    control-plane,master   19h   v1.26.3+b404935
ip-10-0-154-21.eu-west-1.compute.internal    Ready    worker                 19h   v1.26.3+b404935
ip-10-0-176-246.eu-west-1.compute.internal   Ready    worker                 19h   v1.26.3+b404935
ip-10-0-179-65.eu-west-1.compute.internal    Ready    control-plane,master   19h   v1.26.3+b404935
ip-10-0-198-130.eu-west-1.compute.internal   Ready    worker                 19h   v1.26.3+b404935
ip-10-0-216-166.eu-west-1.compute.internal   Ready    control-plane,master   19h   v1.26.3+b404935


$ oc get mcp
NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-abd56ebc6830245cbef4ecff1aaf6e15               True      False      False      3              3                   3                     0                      19h
worker               rendered-worker-caf389063560671ffaea78c3a35e05f7               False     True       False      3              0                   0                     0                      19h
worker-config-test   rendered-worker-config-test-caf389063560671ffaea78c3a35e05f7   True      False      False      0              0                   0                     0                      22m


$ oc get mcp
NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-abd56ebc6830245cbef4ecff1aaf6e15               True      False      False      3              3                   3                     0                      19h
worker               rendered-worker-caf389063560671ffaea78c3a35e05f7               True      False      False      3              3                   3                     0                      19h
worker-config-test   rendered-worker-config-test-caf389063560671ffaea78c3a35e05f7   True      False      False      0              0                   0                     0                      23m
```

## You can remove all the temporary files
Removing the Machine config for the test pool and the test machine pool

```bash
oc delete -f yaml/testing-worker-ntp.yaml
oc delete -f yaml/machineconfigpool.yaml
```

```bash
$ oc get mc
NAME                                               GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
00-master                                          1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
00-worker                                          1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-master-container-runtime                        1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-master-kubelet                                  1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-worker-container-runtime                        1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
01-worker-kubelet                                  1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
99-master-generated-registries                     1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
99-master-ssh                                                                                 3.2.0             20h
99-worker-chrony                                                                              3.2.0             4m5s
99-worker-generated-registries                     1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
99-worker-ssh                                                                                 3.2.0             20h
rendered-master-154d86fafe809ecf601542f6d428c0ee   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             57m
rendered-master-85d1c14b374db56103ef04a6985b3638   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
rendered-master-abd56ebc6830245cbef4ecff1aaf6e15   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             55m
rendered-master-f7e23a7d6b8d6c47a48f6fb4e0080f06   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
rendered-worker-12a3611d04fb7bb004569b75c3ae9bc0   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             55m
rendered-worker-282067975d53d79608eab4edffc6ddbe   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h
rendered-worker-472b63d4d46ed3a2b648633cfd32a96c   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             57m
rendered-worker-caf389063560671ffaea78c3a35e05f7   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             4m
rendered-worker-e7736477cda652d372684230110b8fa2   1ae3805822dfee3263bfd553d49fd36c648fbd12   3.2.0             19h


$ oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-abd56ebc6830245cbef4ecff1aaf6e15   True      False      False      3              3                   3                     0                      19h
worker   rendered-worker-caf389063560671ffaea78c3a35e05f7   True      False      False      3              3                   3                     0                      19h
```

## License

This work is under [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).

Please read the [LICENSE](LICENSE) file for more details.
