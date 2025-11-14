# Manual-Loadbalancing-Wildfly
Manual-Loadbalancing-Wildfly

# WildFly HA Cluster with Load Balancer

This repository contains the configuration and setup instructions for a **WildFly High Availability (HA) cluster** with a **load balancer**.

## Overview

| Component     | Role                                                                                                                                                                                                                                      |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Node1 & Node2 | WildFly standalone servers running `standalone-full-ha.xml`. Host applications (e.g., `Xe.war`) and participate in clustering via JGroups and Infinispan.                                                                                 |
| Load Balancer | WildFly server running `standalone-load-balancer.xml`. Handles incoming requests and distributes them to Node1 & Node2 using mod_cluster.                                                                                                 |
| Config Files  | - `standalone-full-ha.xml` → Full HA profile with clustering. <br> - `standalone-load-balancer.xml` → mod_cluster load balancer config. <br> - `standalone.conf.bat` → JVM options, memory settings, and system properties for each node. |

## Node1 & Node2 Configuration

### standalone-full-ha.xml

**Purpose:** Enables **high availability and clustering**.

**Key Sections:**

* **JGroups Subsystem** – Handles cluster communication between nodes via multicast or TCP.
* **Infinispan Subsystem** – Ensures HTTP session replication and distributed caching.
* **HTTP / AJP Connectors** – Nodes use AJP (or HTTP) to register with the load balancer.
* **Deployment Registration** – Applications (WARs) deployed on one node are replicated across the cluster.

### standalone.conf.bat (Environment Setup)

Sets Java options, memory, and system properties:

```bat
set "JAVA_OPTS=-Xms1024m -Xmx4096m -Djava.net.preferIPv4Stack=true"
set "JAVA_OPTS=%JAVA_OPTS% -Djboss.node.name=node1 -Djboss.bind.address=192.168.200.171 -Djboss.bind.address.private=0.0.0.0 -Djboss.socket.binding.port-offset=0 -Djboss.tx.node.id=node1"
```

Node2 example:

```bat
set "JAVA_OPTS=%JAVA_OPTS% -Djboss.node.name=node2 -Djboss.bind.address=192.168.200.172 -Djboss.socket-binding.port-offset=100 -Djboss.tx.node.id=node2"
```

---

## Load Balancer Configuration

### standalone-load-balancer.xml

**Purpose:** Distributes traffic across Node1 and Node2 using **mod_cluster**.

**Key Sections:**

* **Mod_cluster Filter** – Ensures single-affinity for session consistency, nodes auto-register via multicast.
* **HTTP Listeners** – Accepts client requests and mod_cluster communication.
* **Interface Binding** – Load balancer listens on the correct IP for client requests.

### Node Registration to Load Balancer

When nodes start, they advertise themselves to the load balancer:

```
Registering node node1, connection: ajp://192.168.200.170:8009
Registering node node2, connection: ajp://192.168.200.170:8109
```

---

## Step-by-Step Deployment Procedure

### Step 1: Prepare Node1 & Node2

1. Copy `standalone-full-ha.xml` and `standalone.conf.bat` to Node1 and Node2.
2. Configure unique bind addresses:

   * Node1: `-Djboss.bind.address=192.168.200.171`
   * Node2: `-Djboss.bind.address=192.168.200.172`
3. Start each node:

```bat
Node1: standalone.bat -c standalone-full-ha.xml -Djboss.node.name=node1 -Djboss.socket.binding.port-offset=0
Node2: standalone.bat -c standalone-full-ha.xml -Djboss.node.name=node2 -Djboss.socket-binding.port-offset=100
```

4. Verify cluster membership in logs:

```
ISPN100010: Finished rebalance with members [node2, node1]
```

### Step 2: Configure and Start Load Balancer

1. Place `standalone-load-balancer.xml` in the load balancer WildFly directory.
2. Configure `standalone.conf.bat` for memory and JVM options.
3. Start load balancer:

```bat
standalone.bat -c standalone-load-balancer.xml
```

4. Verify nodes are registered:

```
Registering node node1
Registering node node2
```

### Step 3: Test Load Balancing

1. Access application via load balancer URL:

```
http://192.168.200.170:8095/Xe
```

2. Verify requests are distributed across Node1 and Node2.
3. Optional: Stop one node to check session failover — sessions persist on the other node.

---

## Notes & Best Practices

* Ensure firewall and multicast traffic are allowed between nodes and load balancer.
* Each node must have a unique `jboss.node.name`.
* Use AJP or HTTP connectors consistently between nodes and load balancer.
* Use `standalone-full-ha.xml` only for cluster nodes; load balancer uses `standalone-load-balancer.xml`.
* Monitor logs for node joins/leaves, cache rebalancing, and session replication.

**Benefits of this setup:**

* High availability for enterprise applications
* Load-balanced client requests
* Session replication between nodes
* Horizontal scalability by adding more nodes

