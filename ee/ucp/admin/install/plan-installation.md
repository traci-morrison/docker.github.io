---
title: Plan your installation
description: Learn about the Docker Universal Control Plane architecture, and the requirements to install it on production.
keywords: UCP, install, Docker EE
---

Docker Universal Control Plane helps you manage your container cluster from a
centralized place. This article explains what you need to consider before
deploying Docker Universal Control Plane for production.

## System requirements

Before installing UCP, make sure that all nodes (physical or virtual
machines) that you'll manage with UCP:

* [Comply with the system requirements](system-requirements.md), and
* Are running the same version of Docker Engine.

## Hostname strategy

Docker UCP requires Docker Enterprise. Before installing Docker Enterprise on
your cluster nodes, you should plan for a common hostname strategy.

Decide if you want to use short hostnames, like `engine01`, or Fully Qualified
Domain Names (FQDN), like `node01.company.example.com`. Whichever you choose,
confirm your naming strategy is consistent across the cluster, because
Docker Engine and UCP use hostnames.

For example, if your cluster has three hosts, you can name them:

```none
node1.company.example.com
node2.company.example.com
node3.company.example.com
```

## Static IP addresses

Docker UCP requires each node on the cluster to have a static IP address.
Before installing UCP, ensure your network and nodes are configured to support
this.

## Avoid IP range conflicts

The following table lists recommendations to avoid IP range conflicts.

| Component  | Subnet                     | Range                                    | Default IP address     |
|------------|----------------------------|------------------------------------------|----------------|
| Engine     | `fixed-cidr`               | CIDR range for `docker0` interface and local containers | 172.17.0.0/16  |
| Engine     | `default-address-pools`    | CIDR range for `docker_gwbridge` interface and bridge networks | 172.18.0.0/16  |
| Swarm      | `default-addr-pool`        | CIDR range for Swarm overlay networks    | 10.0.0.0/8     |
| Kubernetes | `pod-cidr`                 | CIDR range for Kubernetes pods           | 192.168.0.0/16 |
| Kubernetes | `service-cluster-ip-range` | CIDR range for Kubernetes services       | 10.96.0.0/16   |


### Engine

There are two IP ranges used by the engine for the `docker0` and `docker_gwbridge` interface:

#### docker0

By default, the Docker engine creates and configures the host system with a network interface called `docker0`, which is an ethernet bridge device. If you don't specify a different network when starting a container, the container is connected to the bridge and all traffic coming from and going to the container flows over the bridge to the Docker engine, which handles routing on behalf of the container.

Docker engine creates `docker0` with a configurable IP range. Containers which are connected to the default bridge are allocated IP addresses within this range. Certain default settings apply to `docker` unless you specify otherwise. The default subnet for `docker0` is `172.17.0.0/16`.

The recommended way to configure the `docker0` settings is to use the `daemon.json` file. You can specify one or more of the following settings to configure the `docker0` interface:

```json
{
  "bip": "172.17.0.1/16",
  "fixed-cidr": "172.17.0.0/16",
}
```

`bip`: Supply a specific bridge IP range for the `docker0` interface, using standard CIDR notation. Default is `172.17.0.1/16`.

`fixed-cidr`: Restrict the IP range for `docker0`, using standard CIDR notation. Default is `172.17.0.0/16`.
This range must be an IPv4 range for fixed IPs, and must be a subset of the bridge IP range (`bip` in `daemon.json`). For example, with `172.17.0.0/17`, IPs for your containers will be chosen from the first half of addresses(`172.17.0.1` - `172.17.127.254`) included in the `bip`(`172.17.0.0/16`) subnet.

#### docker_gwbridge

 The `docker_gwbridge` is a virtual bridge that connects the overlay networks (including the `ingress` network) to an individual Docker engine's physical network. Docker creates it automatically when you initialize a swarm or join a Docker host to a swarm, but it is not a Docker device. It exists in the kernel of the Docker host. The default subnet for `docker_gwbridge` is `172.18.0.0/16`.

 > Note
 >
 > If you need to customize the `docker_gwbridge` settings, you must do so before joining the host to the swarm, or after temporarily removing the host from the swarm.

 The recommended way to configure the `docker_gwbridge` settings is to use the `daemon.json` file. You can specify one or more of the following settings to configure the interface:

 ```json
 {
     "default-address-pools": [
         {"base":"172.18.0.0/16","size":24},
         {"base":"###.###.###.###/##","size":##}
     ]
 }
 ```

`default-address-pools`:  A list of IP address pools for local bridge networks, the default is a single pool `{"base":"172.18.0.0/16","size":24}`. This allocates `/24` network from the `172.18.0.0/16` CIDR range for local bridge networks. Each entry in the list contain the following:

`base`: CIDR range to be divided up for bridge networks, the default is `172.18.0.0/16`

`size`: CIDR netmask that determines the default network size to allocate from the `base` pool, the default is `24`

### Swarm

Swarm uses a default address pool of `10.0.0.0/8` for its overlay networks. If this conflicts with your current network implementation, please use a custom IP address pool. To specify a custom IP address pool, use the `--default-addr-pool` command line option during [Swarm initialization](../../../../engine/swarm/swarm-mode.md).

> Note
>
> The Swarm `default-addr-pool` setting is separate from the Docker engine `default-address-pools` setting. They are two separate ranges that are used for different purposes.

> Note
> 
> Currently, the UCP installation process does not support this flag. To deploy with a custom IP pool, Swarm must first be initialized using this flag and UCP must be installed on top of it.

### Kubernetes

There are two internal IP ranges used within Kubernetes that may overlap and
conflict with the underlying infrastructure:

* The Pod Network -  Each Pod in Kubernetes is given an IP address from either
  the Calico or Azure IPAM services. In a default installation Pods are given
  IP addresses on the `192.168.0.0/16` range. This can be customized at install
  time using the `--pod-cidr` flag.
* The Services Network - When a user exposes a Service in Kubernetes it is
  accessible via a VIP, this VIP comes from a Cluster IP Range. By default on UCP
  this range is `10.96.0.0/16`. Beginning with 3.1.8, this value can be
  changed at install time with the `--service-cluster-ip-range` flag.

## Avoid firewall conflicts

For SUSE Linux Enterprise Server 12 SP2 (SLES12), the `FW_LO_NOTRACK` flag is turned on by default in the openSUSE firewall. This speeds up packet processing on the loopback interface, and breaks certain firewall setups that need to redirect outgoing packets via custom rules on the local machine.

To turn off the FW_LO_NOTRACK option, edit the `/etc/sysconfig/SuSEfirewall2` file and set `FW_LO_NOTRACK="no"`. Save the file and restart the firewall or reboot.

For SUSE Linux Enterprise Server 12 SP3, the default value for `FW_LO_NOTRACK` was changed to `no`.

For Red Hat Enterprise Linux (RHEL) 8, if firewalld is running and `FirewallBackend=nftables` is set in `/etc/firewalld/firewalld.conf`, change this to `FirewallBackend=iptables`, or you can explicitly run the following commands to allow traffic to enter the default bridge (docker0) network:

```
firewall-cmd --permanent --zone=trusted --add-interface=docker0
firewall-cmd --reload
```
## Time synchronization

In distributed systems like Docker UCP, time synchronization is critical
to ensure proper operation. As a best practice to ensure consistency between
the engines in a UCP cluster, all engines should regularly synchronize time
with a Network Time Protocol (NTP) server. If a server's clock is skewed,
unexpected behavior may cause poor performance or even failures.

## Load balancing strategy

Docker UCP doesn't include a load balancer. You can configure your own
load balancer to balance user requests across all manager nodes.

If you plan to use a load balancer, you need to decide whether you'll
add the nodes to the load balancer using their IP addresses or their FQDNs.
Whichever you choose, be consistent across nodes. When this is decided,
take note of all IPs or FQDNs before starting the installation.

[Learn how to set up your load balancer](../configure/join-nodes/use-a-load-balancer.md).

## Load balancing UCP and DTR

By default, UCP and DTR both use port 443. If you plan on deploying UCP and
DTR, your load balancer needs to distinguish traffic between the two by IP
address or port number.

* If you want to configure your load balancer to listen on port 443:
  * Use one load balancer for UCP and another for DTR.
  * Use the same load balancer with multiple virtual IPs.
* Configure your load balancer to expose UCP or DTR on a port other than 443.

If you want to install UCP in a high-availability configuration that uses
a load balancer in front of your UCP controllers, include the appropriate IP
address and FQDN of the load balancer's VIP by using
one or more `--san` flags in the [install command](/reference/ucp/3.0/cli/install.md)
or when you're asked for additional SANs in interactive mode.
[Learn about high availability](../configure/set-up-high-availability.md).

## Use an external Certificate Authority

You can customize UCP to use certificates signed by an external Certificate
Authority. When using your own certificates, you need to have a certificate
bundle that has:

* A ca.pem file with the root CA public certificate,
* A cert.pem file with the server certificate and any intermediate CA public
certificates. This certificate should also have SANs for all addresses used to
reach the UCP manager,
* A key.pem file with server private key.

You can have a certificate for each manager, with a common SAN. For
example, on a three-node cluster, you can have:

* node1.company.example.org with SAN ucp.company.org
* node2.company.example.org with SAN ucp.company.org
* node3.company.example.org with SAN ucp.company.org

You can also install UCP with a single externally-signed certificate
for all managers, rather than one for each manager node. In this case,
the certificate files are copied automatically to any new
manager nodes joining the cluster or being promoted to a manager role.

## Where to go next

* [System requirements](system-requirements.md)
* [Install UCP](index.md)

