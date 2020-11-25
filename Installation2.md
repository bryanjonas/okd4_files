https://www.youtube.com/watch?v=d03xg2PKOPg
## Services VM
The next step is to create a VM that will host several of the services necessary for  establishing our cluster. I chose to
use CentOS to complement my Red Hat styling for the cluster (and because that's what the tutorial used). As opposed to the 
tutorial, however, I chose to only hook up the cluster internal virtual bridge to this machine. 


Two interfaces:
ens18 - external network - 192.168.100.1
ens19 - internal network - 10.10.10.1 - "use only for resource on its network"

Set an internal and external firewall zone for each of the connections
```{bash}
nmcli connection modify ens18 connection.zone external
nmcli connection modify ens19 connection.zone internal
```

Add masquerade to both zones ***Look up***
```{bash}
firewall-cmd --zone=external --add-masquerade --permanent
firewall-cmd --zone=internal --add-masquerade --permanent

```
### DNS Server
This first thing to install is *bind* as the DNS server for the cluster internal communication. 

```{bash}
sudo dnf -y install bind bind-utils git
git clone https://github.com/bryanjonas/okd4_files
```
I moved a couple files from this cloned repository into position before starting the DNS server. *Note:* My home network requires the use
of my PiHole DNS servers so I've got those as the forwarders in the **named.conf** file. 

```{bash}
sudo cp named.conf /etc/named.conf
sudo cp named.conf.local /etc/named/
sudo mkdir /etc/named/zones
sudo cp db* /etc/named/zones

sudo systemctl enabled named
sudo systemctl start named
firewall-cmd --permanent --zone=internal --add-port=53/udp
firewall-cmd --reload
```

Update the ens18 (external) to use 127.0.0.1 for LAN

Install DHCP Server
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
The next thing that the services will provide is a webserver to host the image and "ignition" files for the CoreOS nodes.

```{bash}
sudo dnf install -y httpd

sudo sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf

sudo setsebool -P httpd_read_user_content 1
sudo systemctl enable httpd
sudo systemctl start httpd
sudo firewall-cmd --permanent --zone=internal --add-port=8080/tcp
sudo firewall-cmd --reload
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
The services VM should now be staged for the build of the nodes.

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

sudo firewall-cmd --permanent --zone=internal --add-service=mountd
sudo firewall-cmd --permanent --zone=internal --add-service=rpc-bind
sudo firewall-cmd --permanent --zone=internal --add-service=nfs
sudo firewall-cmd --reload
```

To get the bootstrap going, you boot the VM and then hit tab when the first CoreOS screen appears so you can add few boot parameters:
```{bash}
coreos.inst.install_dev=/dev/sda 
coreos.inst.image_url=http://10.10.10.2:8080/okd4/fcos.raw.xz 
coreos.inst.ignition_url=http://10.10.10.2:8080/okd4/bootstrap.ign
```
