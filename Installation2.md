# Establishing a OKD Cluster

These instructions are a combination of instructions taken from:
* https://itnext.io/guide-installing-an-okd-4-5-cluster-508a2631cbee
* https://www.youtube.com/watch?v=d03xg2PKOPg

## Services VM (okd4-services)
The first step is to create a VM that will host several of the services necessary for establishing our cluster. I chose to
use CentOS to complement my Red Hat styling for the cluster (and because that's what the tutorial used). 

This VM will have to NICs: one for external communication and one for cluster-internal communication. Some important settings for the interfaces:

**External** (ens18):
* IPv4 Address: 192.168.1.100 (Home LAN reservation)
* Gateway: 192.168.1.1
* DNS: 127.0.0.1 (Wait to set this until DNS server is running)

**Internal** (ens19):
* IPv4 Address: 10.10.10.1 
* Gateway: 10.10.10.1
* DNS: 127.0.0.1
* "Use this connection only for resources on its network"

### Intial Firewall Configuration
We want to establish two firewall zones to go along with our two networks. It should be clear which interface is associated with each zone.
```{bash}
sudo nmcli connection modify ens18 connection.zone external
sudo nmcli connection modify ens19 connection.zone internal

#Check settings
sudo firewall-cmd --get-active-zones
```
In order to allow clients on the internal network to communicate with outside (to grab images and other tasks), we turn on masquerading on 
both interfaces and check to ensure that IP forwarding is turned on.
```{bash}
firewall-cmd --zone=external --add-masquerade --permanent
firewall-cmd --zone=internal --add-masquerade --permanent

#Should print '1' to screen
cat /proc/sys/net/ipv4/ip_forwarding
```

### Download Config Files
We'll need the configuration files included in this repository.

```{bash}
sudo dnf install -y git
git clone https://github.com/bryanjonas/okd4_files
```

### DNS Server
This next thing to install is *bind* as the DNS server for the cluster internal communication. 

```{bash}
sudo dnf -y install bind bind-utils
```

We then move a couple files from our cloned repository into position before starting the DNS server. *Note:* My home network requires the use
of my PiHole DNS servers so I've got those as the forwarders in the **named.conf** file. 

```{bash}
sudo cp named.conf /etc/named.conf
sudo cp named.conf.local /etc/named/
sudo mkdir /etc/named/zones
sudo cp db* /etc/named/zones

sudo firewall-cmd --permanent --zone=internal --add-port=53/udp
sudo firewall-cmd --reload
sudo systemctl enabled named
sudo systemctl start named
```

*Remember to update the ens18 (external) to use 127.0.0.1 for DNS.*

### DHCP Server
We need to provide a DHCP server for the internal network clients. Be sure that the MAC addresses in the configuration file match those of your
clients. 

```{bash}
sudo dnf install -y dhcp-server 
sudo cp dhcpd.conf /etc/dhcp/dhcpd.conf
sudo firewall-cmd --permanent --zone=internal --add-service=dhcp
sudo firewall-cmd --reload
sudo systemctl enable dhcpd
sudo systemctl starrt dhcpd
```

### Loadbalancer
We are going to use HAProxy as the load balancer between the control planes and between the compute nodes. The services VM will act as the load balancer and redirect requests to the various APIs to the proper nodes.

We need to install 
```{bash}
sudo dnf install haproxy -y
sudo cp haproxy.cfg /etc/haproxy/haproxy.cfg

sudo setsebool -P haproxy_connect_any 1
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy

sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=22623/tcp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### Webserver
The next thing that the services VM will provide is a webserver to host the image and "ignition" files for the CoreOS nodes.

```{bash}
sudo dnf install -y httpd

sudo sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf

sudo setsebool -P httpd_read_user_content 1
sudo firewall-cmd --permanent --zone=internal --add-port=8080/tcp
sudo firewall-cmd --reload
sudo systemctl enable httpd
sudo systemctl start httpd
```

The tools for building the node configurations are downloaded from the OKD Release page (https://github.com/openshift/okd/releases). 
Be sure to grab the *client* and *install* files, untar, and move them into the **/usr/local/bin** directory.

After generating a SSH key, the public key should be pasted into the **install_config.yaml** file included in the OKD files from this
repo and that file should be moved into a new directory called **install_dir** in the home directory. Once this is done, you have to back
up the **yaml** or it will be eaten by the next step.

```{bash}
cp install_dir/install-config.yaml install_dir/install-config.yaml.bak

openshift-install create manifests --dir=install_dir/

#If you don't want pods scheduled on the control planes you have to change the manifest for the scheduler using:
sed -i 's/mastersSchedulable: true/mastersSchedulable: False/' install_dir/manifests/cluster-scheduler-02-config.yml

openshift-install create ignition-configs --dir=install_dir/

sudo mkdir /var/www/html/okd4
sudo cp -R install_dir/* /var/www/html/okd4
```

The next step is to download the latest bare-metal images of Fedora CoreOS (https://getfedora.org/coreos/download?tab=metal_virtualized&stream=stable).
We wanted the **raw.xz** version and it's signature (**raw.xz.sig**). I downloaded these and changed shortened their name for ease of use. We want to move
those onto the webserver so our nodes can access them.

```{bash}
sudo mv Downloads/fcos.raw.xz /var/www/html/
sudo mv Download/fcos.raw.xz.sign /var/www/html/

sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html/
```

### Set up Registry Share

```{bash}
sudo dnf install -y nfs-utils
sudo systemctl enable nfs-server rpcbind
sudo systemctl start nfs-server rpcbind
sudo mkdir /shares
sudo mkdir /shares/registry
sudo chown -R nobody:nogroup /shares/registry
sudo chmod -R 777 /shares/registry
echo "/shares/registry 10.10.10.1/24(rw,sync,root_squash,no_subtree_check,no_wdelay)" | sudo tee /etc/exports

sudo setsebool -P nfs_export_all_rw 1
sudo systemctl restart nfs-server
sudo firewall-cmd --permanent --zone=internal --add-service=mountd
sudo firewall-cmd --permanent --zone=internal --add-service=rpc-bind
sudo firewall-cmd --permanent --zone=internal --add-service=nfs
sudo firewall-cmd --reload
```
The services VM should now be staged for the build of the nodes.

## Setting up the Bootstrap, Control Planes, and Workers
The basic cluster is supposed to have one bootstrap, three control planes, and two compute nodes. For the purposes of building the cluster in Proxmox,
I have all the specs the same for the machines: 4 cores, 16 GB RAM and 120 GB of storage.

The DHCP server provides these IP addresses to our nodes:
| IP | Node |
|----|------|
| 10.10.10.2 | okd4-services |
| 10.10.10.200 | okd4-bootstrap |
| 10.10.10.201 | okd4-control-plane-1 |
| 10.10.10.202 | okd4-control-plane-2 |
| 10.10.10.203 | okd4-control-plane-3 |
| 10.10.10.204 | okd4-compute-1 |
| 10.10.10.205 | okd4-compute-2 |

To get the bootstrap going, you boot the VM and then hit tab when the first CoreOS screen appears so you can add few boot parameters:
```{bash}
coreos.inst.install_dev=/dev/sda 
coreos.inst.image_url=http://10.10.10.1:8080/okd4/fcos.raw.xz 
coreos.inst.ignition_url=http://10.10.10.1:8080/okd4/bootstrap.ign
```

The parameters for the control planes are nearly the same:
```{bash}
coreos.inst.install_dev=/dev/sda 
coreos.inst.image_url=http://10.10.10.1:8080/okd4/fcos.raw.xz 
coreos.inst.ignition_url=http://10.10.10.1:8080/okd4/master.ign
```

As are the parameters for the workers:
```{bash}
coreos.inst.install_dev=/dev/sda 
coreos.inst.image_url=http://10.10.10.1:8080/okd4/fcos.raw.xz 
coreos.inst.ignition_url=http://10.10.10.1:8080/okd4/worker.ign
```

It's best to start the bootstrap, the control planes and then the workers. Each node will boot and then download the image from our webserver, monitor the process until you see this download. Chances are good if you get there (and everything else is typed in correctly) that the process will succeed.

You can use this command (on the services VM) to monitor the build process but I find that it is best to watch the HAProxy stats page (http://10.10.10.1:9000) for the clients to come up (and go down and come up again).

```{bash}
openshift-install --dir=install_dir/ wait-for bootstrap-complete --log-level=info
```

Once the control planes are up you will get a message that the bootstrap VM should be taken down. You will also see it go dark on the HAProxy stats page. You can comment out the corresponding entries in the HAProxy configuration file and power off the VM.

The worker nodes won't come up until you approve "certificate signing requests" (csr) so you need to use the following commands to see and approve these.
```{bash}
export KUBECONFIG=~/install_dir/auth/kubeconfig
oc whoami
oc get nodes

oc get csr
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
```

You can watch the **get nodes** screen for the workers to come on line. Check the **get csr** list for any new approvals that are necessary. Once all the nodes are online you can check the HAProxy stats to ensure that all servers are reachable on all APIs.

## Image Registry
We need to provide storage to the Image Registry Operator on the NFS share of our service machine. We've already prepped the services VM side so we need to use the Registry PV manifest included in this repository.

```{bash}
oc create -f okd4_files/registry_pv.yaml
oc get pv
```

We then open the Image Registry Operator and change a couple thing in the file (see below):
```{bash}
#You can add this line if you're a noob like me
export OC_EDITOR="nano"

oc edit configs.imageregistry.operator.openshift.io
```
```{bash}
managementState: Managed (was Removed)

storage:
    pvc:     (These two
      claim:    lines are added)
```

## Access the Interface
You can access the interface at: https://console-openshift-console.apps.lab.okd.local/

If you're watching the *openshift-install* logging of the cluster install you should see this link as well as the username and password pop out at the end. If not, you can get the password from the **~/install_dir/auth/kubeadmin-password** file.

Enjoy!
