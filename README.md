# Install OpenStack

### 添加模型

```bash
juju add-model -c maas-controller --config default-series=focal openstack-lab
```

### Ceph OSD

**ceph-osd.yaml**

```yaml
ceph-osd:
  osd-devices: /dev/sdb
  source: cloud:focal-wallaby
```

```bash
juju deploy --config ceph-osd.yaml --constraints tags=lab ceph-osd
```

### Nova Compute计算节点

**nova-compute.yaml**

```yaml
nova-compute:
  config-flags: default_ephemeral_format=ext4
  enable-live-migration: true
  enable-resize: true
  migration-auth-type: ssh
  openstack-origin: cloud:focal-wallaby
```

```bash
juju deploy -n 1 --to 0 --config nova-compute.yaml nova-compute
```

### Mysql集群
```bash
juju deploy -n 3 --to lxd:0,lxd:0,lxd:0 mysql-innodb-cluster
```

### vault
```bash
juju deploy --to lxd:0 vault

juju deploy mysql-router vault-mysql-router
juju add-relation vault-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation vault-mysql-router:shared-db vault:shared-db
```

------

### Vault初始化解封

```bash
sudo snap install vault

#vault addresss
export VAULT_ADDR="http://10.10.100.186:8200"
vault operator init -key-shares=5 -key-threshold=3

Unseal Key 1: 6Wb8ur1jfNIPuOrDnzU+PGW2aewEvlLxeHtVl1BO3AmR
Unseal Key 2: wTLze7hcbTS2/oXQBLCEMZy/ixTICQg52GC43yY/kNaI
Unseal Key 3: QogDI9yV46yuj+us2zJ8+S5W2QhecwhK0hnaKNP3VWye
Unseal Key 4: yRYsDMVk/gDpIaHJ2PdvoG8RLwrxY/biWux93ZO42As2
Unseal Key 5: LwE745yAIAIcqN6mzkMjZ3qE2rEz8Udz/ivbLJXPAZUi

Initial Root Token: s.EF05wHeZ4wvpBrSbOA5rOJxF

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.


vault operator unseal 6Wb8ur1jfNIPuOrDnzU+PGW2aewEvlLxeHtVl1BO3AmR
vault operator unseal wTLze7hcbTS2/oXQBLCEMZy/ixTICQg52GC43yY/kNaI
vault operator unseal QogDI9yV46yuj+us2zJ8+S5W2QhecwhK0hnaKNP3VWye

export VAULT_TOKEN=s.EF05wHeZ4wvpBrSbOA5rOJxF
vault token create -ttl=10m

Key                  Value
---                  -----
token                s.beJV2JCFw3rTWRdkDMwUNgO0
token_accessor       QwPjjgxHO9G2uR5hQ3bpa9nE
token_duration       10m
token_renewable      true
token_policies       ["root"]
identity_policies    []
policies             ["root"]

juju run-action --wait vault/leader authorize-charm token=
```

```bash
juju add-relation mysql-innodb-cluster:certificates vault:certificates
```

### Neutron网络节点

**neutron.yaml**

```yaml
ovn-chassis:
  bridge-interface-mappings: br-ex:eno2 #eno2为物理网卡名称，且在maas中为未配置状态（Unconfigured）
  ovn-bridge-mappings: physnet1:br-ex	#physnet1为创建的网卡名称，后续添加网络的时候需要用到
neutron-api:
  neutron-security-groups: true
  flat-network-providers: physnet1
  worker-multiplier: 0.25
  openstack-origin: cloud:focal-wallaby
ovn-central:
  source: cloud:focal-wallaby
```

```bash
juju deploy -n 3 --to lxd:0,lxd:0,lxd:0 --config neutron.yaml ovn-central
juju deploy --to lxd:0 --config neutron.yaml neutron-api
juju deploy neutron-api-plugin-ovn
juju deploy --config neutron.yaml ovn-chassis
```

```bash
juju add-relation neutron-api-plugin-ovn:neutron-plugin neutron-api:neutron-plugin-api-subordinate
juju add-relation neutron-api-plugin-ovn:ovsdb-cms ovn-central:ovsdb-cms
juju add-relation ovn-chassis:ovsdb ovn-central:ovsdb
juju add-relation ovn-chassis:nova-compute nova-compute:neutron-plugin
juju add-relation neutron-api:certificates vault:certificates
juju add-relation neutron-api-plugin-ovn:certificates vault:certificates
juju add-relation ovn-central:certificates vault:certificates
juju add-relation ovn-chassis:certificates vault:certificates

juju deploy mysql-router neutron-api-mysql-router
juju add-relation neutron-api-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation neutron-api-mysql-router:shared-db neutron-api:shared-db
```

### Keystone

**keystone.yaml**

```yaml
keystone:
  worker-multiplier: 0.25
  openstack-origin: cloud:focal-wallaby
```

```bash
juju deploy --to lxd:0 --config keystone.yaml keystone

juju deploy mysql-router keystone-mysql-router
juju add-relation keystone-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation keystone-mysql-router:shared-db keystone:shared-db

juju add-relation keystone:identity-service neutron-api:identity-service
juju add-relation keystone:certificates vault:certificates
```

### RabbitMQ

```
juju deploy --to lxd:0 rabbitmq-server
juju add-relation rabbitmq-server:amqp neutron-api:amqp
juju add-relation rabbitmq-server:amqp nova-compute:amqp
```

------

### Nova cloud controller

**nova-cloud-controller.yaml**

```yaml
nova-cloud-controller:
  network-manager: Neutron
  worker-multiplier: 0.25
  openstack-origin: cloud:focal-wallaby
```

```bash
juju deploy --to lxd:0 --config nova-cloud-controller.yaml nova-cloud-controller

juju deploy mysql-router ncc-mysql-router
juju add-relation ncc-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation ncc-mysql-router:shared-db nova-cloud-controller:shared-db

juju add-relation nova-cloud-controller:identity-service keystone:identity-service
juju add-relation nova-cloud-controller:amqp rabbitmq-server:amqp
juju add-relation nova-cloud-controller:neutron-api neutron-api:neutron-api
juju add-relation nova-cloud-controller:cloud-compute nova-compute:cloud-compute
juju add-relation nova-cloud-controller:certificates vault:certificates
```

### Placement

**placement.yaml**

```yaml
placement:
  worker-multiplier: 0.25
  openstack-origin: cloud:focal-wallaby
```

```bash
juju deploy --to lxd:0 --config placement.yaml placement

juju deploy mysql-router placement-mysql-router
juju add-relation placement-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation placement-mysql-router:shared-db placement:shared-db

juju add-relation placement:identity-service keystone:identity-service
juju add-relation placement:placement nova-cloud-controller:placement
juju add-relation placement:certificates vault:certificates
```

### OpenStack dashboard

```bash
juju deploy --to lxd:0 --config openstack-origin=cloud:focal-wallaby openstack-dashboard

juju deploy mysql-router dashboard-mysql-router
juju add-relation dashboard-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation dashboard-mysql-router:shared-db openstack-dashboard:shared-db

juju add-relation openstack-dashboard:identity-service keystone:identity-service
juju add-relation openstack-dashboard:certificates vault:certificates
```

### Glance

**glance.yaml**

```yaml
glance:
  worker-multiplier: 0.25
  openstack-origin: cloud:focal-wallaby
```

```bash
juju deploy --to lxd:0 --config glance.yaml glance

juju deploy mysql-router glance-mysql-router
juju add-relation glance-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation glance-mysql-router:shared-db glance:shared-db

juju add-relation glance:image-service nova-cloud-controller:image-service
juju add-relation glance:image-service nova-compute:image-service
juju add-relation glance:identity-service keystone:identity-service
juju add-relation glance:certificates vault:certificates
```

------

### Ceph monitor

**ceph-mon.yaml**

```yaml
ceph-mon:
  expected-osd-count: 1
  monitor-count: 1
  source: cloud:focal-wallaby
```

```
juju deploy --to lxd:0 --config ceph-mon.yaml ceph-mon

juju add-relation ceph-mon:osd ceph-osd:mon
juju add-relation ceph-mon:client nova-compute:ceph
juju add-relation ceph-mon:client glance:ceph
```

### Cinder

**cinder.yaml**

```yaml
cinder:
  block-device: None
  glance-api-version: 2
  worker-multiplier: 0.25
  openstack-origin: cloud:focal-wallaby
```

```bash
juju deploy --to lxd:0 --config cinder.yaml cinder

juju deploy mysql-router cinder-mysql-router
juju add-relation cinder-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation cinder-mysql-router:shared-db cinder:shared-db

juju add-relation cinder:cinder-volume-service nova-cloud-controller:cinder-volume-service
juju add-relation cinder:identity-service keystone:identity-service
juju add-relation cinder:amqp rabbitmq-server:amqp
juju add-relation cinder:image-service glance:image-service
juju add-relation cinder:certificates vault:certificates

juju deploy cinder-ceph

juju add-relation cinder-ceph:storage-backend cinder:storage-backend
juju add-relation cinder-ceph:ceph ceph-mon:client
juju add-relation cinder-ceph:ceph-access nova-compute:ceph-access
```

### Ceph RADOS Gateway

```bash
juju deploy --to lxd:0 --config source=cloud:focal-wallaby ceph-radosgw
juju add-relation ceph-radosgw:mon ceph-mon:radosgw
```

### NTP

```bash
juju deploy ntp
juju add-relation ceph-osd:juju-info ntp:juju-info
```

------

# Final results and dashboard access

```bash
juju status --format=yaml openstack-dashboard | grep public-address | awk '{print $2}' | head -1

#开启控制台访问
juju config nova-cloud-controller console-access-protocol=novnc
```
