## Adding a Node

After building a new VM matching the specs of the previous machines, you need to add the MAC address to the DHCP server, DNS records, and the HAProxy.

### DHCP Server
```{bash}
sudo nano /etc/dhcp/dhcpd.conf
sudo systemctl restart dhcpd
```

### DNS Records
```{bash}
sudo nano /etc/named/zones/db.10.10.10.10
#Add line for new machine at bottom

sudo nano /etc/named/zones/db.okd.local
#Add line under A records for OCP Cluster

sudo systemctl restart named
```

### HAProxy
```{bash}
sudo nano /etc/haproxy/haproxy.cfg
#Add line(s) for appropriate backends

sudo systemctl restart haproxy
```

### Approve CSR
As with the original nodes, a series of certificate signing requests will come through that must be approved.
```{bash}
#From the Services VM
oc get csr

oc set csr -o name | xargs oc adm certificate approve
```
