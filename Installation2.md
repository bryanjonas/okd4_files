https://www.youtube.com/watch?v=d03xg2PKOPg

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

```{bash}
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
```
