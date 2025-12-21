---
title: "Change durable queue type of RabbitMQ on Kolla-Ansible"
date: 2025-01-20
author: "Aidil Putra"

summary: >
  Changes RabbitMQ HA type before upgrade Openstack Kolla-Ansible
  from 2024.2 to 2025.1

tags:
  - openstack
  - kolla-ansible
  - rabbitmq

coverAlt: >
  A nighttime scene at Genting Highlands, Pahang, Malaysia,
  showing tall buildings glowing with warm lights against
  the cool darkness of the surrounding highlands."

coverCaption: >
  Genting Highlands, Pahang, Malaysia. Photo © Aidil Putra, August 2025.

---

## Introduction

I’ve been planning an upgrade of our OpenStack environment from **2024.2** to **2025.1**.
At first glance, it looked like a routine release upgrade. However, once I started
reading the Kolla-Ansible documentation more carefully, it became clear that this
upgrade requires **proper preparation**, especially around **RabbitMQ**.

This post is mainly a personal note while preparing the upgrade, but hopefully it
can also help other operators running OpenStack.

In the **Epoxy** release, Kolla-Ansible upgrades **RabbitMQ to version 4.0**.
Because of this change, **all queues must be migrated to durable queue types**
*before* performing the upgrade.

If you previously enabled `om_enable_rabbitmq_high_availability` then we need to
migrate to `om_enable_rabbitmq_quorum_queues`, because of this changes we need to remove
the old transient queues and exchanges by stop all related openstack services and perform
Reset the state on each RabbitMQ.

---

## Configuration

To use `Quorum queues` we need set configuration like this on our `/etc/kolla/globals.yml`:

```yaml
om_enable_queue_manager: true
om_enable_rabbitmq_quorum_queues: true
om_enable_rabbitmq_transient_quorum_queue: true
om_enable_rabbitmq_stream_fanout: true
```

---

## Reconfigure

> **Warning!** Ensure your RabbitMQ cluster is in a healthy state before performing any of these steps.

### Get related services
We need to get related services that effected of this rabbitmq changes

```bash
fgrep -l -r 'transport_url =' /etc/kolla | cut -d- -f1 \
| cut -d\/ -f4 | sort |uniq -c | awk '{ print $2 }'
```

Output:

```bash
aodh
barbican
ceilometer
cinder
cloudkitty
designate
glance
heat
ironic
keystone
neutron
nova
octavia
```

### Stop Openstack services
Stop all OpenStack services that interact with RabbitMQ so they do not attempt to
recreate queues prematurely:

```bash
kolla-ansible stop -t aodh,barbican,ceilometer,cinder,cloudkitty, \
designate,glance,heat,ironic,keystone,neutron,nova,octavia \
--yes-i-really-really-mean-it
```


### Reconfigure RabbitMQ

If RabbitMQ high availability was previously enabled, reconfigure RabbitMQ to apply
the new settings:

```bash
kolla-ansible reconfigure --tags rabbitmq
```

---

### Reset RabbitMQ state

Reset the state on each RabbitMQ node to remove old transient queues and exchanges:

```bash
kolla-ansible rabbitmq-reset-state
```

---

### Start OpenStack services

Start the OpenStack services so they can recreate the queues as **durable queues**:

```bash
kolla-ansible deploy -t aodh,barbican,ceilometer,cinder,cloudkitty, \
designate,glance,heat,ironic,keystone,neutron,nova,octavia

```

---

## Refrences

- https://docs.openstack.org/kolla-ansible/2024.1/reference/message-queues/rabbitmq.html
