Based on the popular tutorial here: https://itnext.io/guide-installing-an-okd-4-5-cluster-508a2631cbee

### Cluster Internal Network
The first step is establishing an internal network (VLAN) for cluster communications. On Proxmox I did this by an 
additional virtual bridge that was unattached to any real interface. I set this CIDR as 10.10.10.1/24 which will be
used for all cluster-internal IPs.

### pfSense VM
I created a fairly low-power VM to install pfSense to utilize as the DHCP server as well as the router between 
external and internal communications. For my particular set-up, the WAN is *vtnet0* and the *vtnet1* for the LAN.

|Interface | Adapter | IP |
|----------|---------|----|
|WAN (wan) | vtnet0 | v4: 192.168.1.100/24|    
|LAN (lan) | vtnet1  | v4: 10.10.10.1/24|

### Services VM
The next step is to create a VM that will host several of the services necessary for  establishing our cluster. I chose to
use CentOS to complement my Red Hat styling for the cluster (and because that's what the tutorial used). As opposed to the 
tutorial, however, I chose to only hook up the cluster internal virtual bridge to this machine. I decided to try to let pfSense
handle the DNS duties instead of the services VM. Call me crazy for wanting the routing software to be in charge of the routing
duties as opposed to something I set up.

### DNS Resolver
I enabled the pfSense DNS resolver and ensured the WAN DNS server address was pointed at my home network DNS server. Following that
I set up several DNS reservations to ensure that APIs and endpoints were resolved to the correct machines.

| Host | Parent Domain | IP |
|------|---------------|----|
| Name Servers | | |
| okd4-service | okd.local | 10.10.10.2 |
| Platform IPs | | |
| okd4-bootstrap | lab.okd.local | 10.10.10.200 |
| okd4-compute-1 | lab.okd.local | 10.10.10.204 |
| okd4-control-plane-1 | lba.okd.local | 10.10.10.201 |
| Cluster IPs | | |
| api | lab.okd.local 10.10.10.2 |
| api-int | lab.okd.local | 10.10.10.2 |
| * | apps.lab.okd.local | 10.10.10.2 |
| etcd-0 | lab.okd.local | 10.10.10.201 |
| console-openshift-console | apps.lab.okd.local | 10.10.10.2 |
| oauth-openshift | apps.lab.okd.local | 10.10.10.2 |

| Domain | IP |
|--------|----|
| apps.lab.okd.local | 10.10.10.2 |

I'm going to leave these records here for later in case I add extra machines.
| Host | Parent Domain | IP |
|------|---------------|----|
| etcd-1 | lab.okd.local | 10.10.10.202 |
| etcd-2 | lab.okd.local | 10.10.10.203 |

### DHCP Reservations
You need to make several DHCP reservations to ensure our various machine end up at particular IPs.

10.10.10.2 -> okd4-services
