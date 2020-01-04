# Single-OpenVPN-Server-to-Handle-Multiple-Client-with-Multiple-Static-Public-IP-on-debian
it is a post installation modification on OpenVPN server to handle multiple client with individual static IP

First, install OpenVPN server and configure necessary certificate for server and client. A nice script to deploy a complete OpenVPN server with client profile, is available at https://github.com/Angristan/OpenVPN-install

Following 2 Dummy IP addresses assumed as public IP
```
Public IP-1: 223.10.10.50 255.255.255.0
Public IP-2: 223.10.11.51 255.255.255.0
```

#These IPs are from different block so need to ensure reachability of 2 IP using command
```
$ ip neigh ;if both ip reachable then output will reveal readable understnading
```

#Assign static private IP address for both OpenVPN client that the connection can be made on fixed private IP every time
```
sudo mkdir /etc/openvpn/ccd   
sudo chown -R nobody:nogroup /etc/openvpn/ccd
sudo nano /etc/openvpn/ccd/Client1
ifconfig-push 10.8.0.101 255.255.255.0  ;save it
sudo nano /etc/openvpn/ccd/Client2
ifconfig-push 10.8.0.102 255.255.255.0  ;save it
sudo nano /etc/openvpn/ipp.txt
Client1,10.8.0.101  ;Pushed to Client1
Client2,10.8.0.102  ;Pushed to Client2
```

#Modify OpenVPN server config to adjust above configuration to be working
```
sudo nano /etc/openvpn/server.conf
client-config-dir /etc/openvpn/ccd  ;add the line in config file
push "redirect-gateway def1" ; bypass-dhcp has been remove form this line
```

#Allow IP Forward
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
echo "net.ipv4.conf.all.forwarding=1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding=1" >> /etc/sysctl.conf
echo "net.ipv4.proxy_arp=1" >> /etc/sysctl.conf
```

#Modify Firewall rules using iptables
```
First need to install "iptables-persistent"
apt-get install iptables-persistent
iptables-save > /etc/iptables/rules.v4 ;save current rules
cd /etc/iptables
ls ;check and delete other files except rules.v4 and rules.v6
sudo nano /etc/iptables/rules.v4  ;modify rules in here
```
#Add the following Rules and remove rules like "-A POSTROUTING -s 10.8.0.1/24 -o eth0 -J MASQUEREDE"
```
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.8.0.101 -o eth0 -j SNAT --to-source 223.10.11.51
-A POSTROUTING -s 10.8.0.102 -o eth0 -j SNAT --to-source 223.10.10.50
-A PREROUTING -i eth0 -d 223.10.10.50 -p udp -j DNAT --to 10.8.0.102:1194
-A PREROUTING -s 10.8.0.102 -p udp --dport 1194 -j DNAT --to-destination 223.10.10.50
COMMIT
```

The following NAT rules will enable "Client1" to have access on IP 223.10.11.51 and "Client2" to have access on IP 223.10.10.50
