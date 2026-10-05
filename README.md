# On-prem to Managed Kafka, by way of a transit VPC

Your data center reaches Google Cloud through a transit VPC. Your Kafka cluster lives behind a Shared VPC. Here is how a producer in the data center finds a broker, and what you configure so it can.

Swipe the map sideways to see the whole route.

## Why the transit VPC gets its own Kafka subnet

Managed Service for Apache Kafka doesn't put brokers in your network. It runs them in a Google-owned tenant project and drops Private Service Connect endpoints, one for the bootstrap address and one per broker, into each subnet you connect to the cluster. It also writes private DNS records for those endpoints into each connected network.

**The catch:** Private Service Connect endpoints aren't reachable across VPC Network Peering. Traffic that arrives from on-prem into the transit VPC can't hop over the peering to the endpoints sitting in the Shared VPC, and the transit VPC never receives the DNS records either.

The fix without Network Connectivity Center is to connect a second subnet, one that lives in the transit VPC, to the same cluster. A cluster accepts up to ten subnets, one per VPC network, all in the cluster's region, and they can come from other projects. The cluster then has two front doors with the same hostnames: one in the Shared VPC for workloads in your service projects, and one in the transit VPC for everything arriving over Interconnect or HA VPN.

Any peering you already have between the transit VPC and the Shared VPC can stay for other traffic. Kafka just doesn't use it.

## Size the Kafka subnets

Each connected subnet needs one address per broker plus one for the bootstrap endpoint. Clusters up to 45 vCPUs run three brokers; above that it's one broker per 15 vCPUs. Google recommends three times the minimum as headroom, and you should size for the biggest the cluster will grow.

Largest planned cluster size (vCPUs)

## Generate the build steps

Fill in your names. The commands below update as you type. Everything runs from a workstation with gcloud and permission on the three projects.

Cluster

Cluster project ID Cluster project number Cluster name Region Starting vCPUs

Shared VPC (host project)

Host project ID Shared VPC network Kafka subnet name Kafka subnet range

Transit VPC

Transit project ID Transit VPC network Kafka subnet name Kafka subnet range Cloud Router name

Clients

Client service account

```

```

## Before you call it done

Your ticks are saved in this browser.

## When it doesn't connect

| What you see | Usual cause | Fix |
| --- | --- | --- |
| Bootstrap name doesn't resolve on-prem | No conditional forwarder, or it points at the wrong network's resolvers. | Forward the Kafka domain to the inbound forwarder addresses in the *transit* VPC, not the Shared VPC. |
| Name resolves to a Shared VPC address, then times out | On-prem is getting answers from the Shared VPC's zone. That endpoint is only reachable over peering, which Kafka can't use. | Point forwarding at the transit VPC so answers come back with transit-subnet addresses. |
| Bootstrap connects, then the client hangs on metadata or produce | Broker hostnames don't resolve, or broker endpoint addresses aren't routed back on-prem. | Check that every broker name resolves to the transit Kafka subnet, and that the whole subnet range is advertised by Cloud Router. |
| Cluster logs "failed to set up networking in VPC subnet" | Service agent lacks the role in the host or transit project, the subnet is out of addresses or in another region, or a second subnet from the same VPC was added. | Grant Managed Kafka Service Agent in both projects, resize the subnet, keep one subnet per VPC in the cluster's region. |
| Works from one data center, not another | The second site's VLAN attachments or tunnels land in a different region from the endpoints. | Terminate hybrid links for Kafka traffic in the cluster's region, or confirm cross-region access to the endpoints before relying on it. |
| TCP connects, authentication fails | Client can't fetch Google tokens from on-prem, or its identity lacks the Managed Kafka Client role. | Allow reach to Google's token endpoints (internet or Private Google Access for on-prem hosts) and grant the role on the cluster project. |

Hostname formats can differ between clusters. Always copy the real bootstrap address from `gcloud managed-kafka clusters describe`; the one shown here is the common pattern.