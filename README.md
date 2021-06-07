<h2>A primer on IPTABLES:</h2>
In linux operating system, the firewalling is taken care of using netfilter. Which is a kernel module that decides what packets are allowed to come in or to go outside.Iptables are just the interface to netfilter.And by firewalling we mean that deciding which packets are allowed to go in/out of the system.

There are three important tables: mangle, filter and nat. 
The mangle table is responsible for the alteration of service bits in the TCP header. 
The filter queue is responsible for packet filtering. 
The nat table performs Network Address Translation (NAT). Each tables may have some built-in chains in which firewall policy rules can be placed.


The filter table has three built-in chains:

   1. Forward chain: Filters packets destined for networks protected by the firewall.
   2. Input chain: Filters packets destined for the firewall.
   3. Output chain: Filters packets originating from the firewall.
    
The nat table has the following built-in chains:

   1. Pre-routing chain: NATs packets when the destination address of the packet needs to be changed.
   2. Post-routing chain: NATs packets when the source address of the packet needs to be changed.
   3. Output chain: NATs packets originating from the firewall.
```    
PACKET IN
    |
PREROUTING--[routing]-->--FORWARD-->--POSTROUTING-->--OUT
 - nat (dst)   |           - filter      - nat (src)
               |                            |
               |                            |
              INPUT                       OUTPUT
              - filter                    - nat (dst)
               |                          - filter
               |                            |
               `----->-----[app]----->------'
 ```           
               
Note: if the packet is from the firewall, it will not go through the PREROUTING chain.
The packet entering the firewall is inspected by the rules in the nat table’s PREROUTING chain to see whether it requires destination modification (DNAT). The packet is then routed by Linux router after leaving the PREROUTING chain. The packet which is destined for a “protected” network is filtered by the rules in the FORWARD chain of the filter table. The it will go through the packet undergoes SNAT in the POSTROUTING chain before arriving at the “protected” network. When the destination server decides to reply, the packet undergoes the same sequence of steps.


Defining Chain Rules

Defining a rule means appending it to the chain. To do this, you need to insert the -A option (Append) right after the iptables command, like so:

```
sudo iptables -A
```

It will alert iptables that you are adding new rules to a chain. Then, you can combine the command with other options, such as:
   * -t (table) - this option specifies the packet matching table it can be either filter , mangle or nat . The default value is filter
   * -i (interface) — the network interface whose traffic you want to filter, such as eth0, lo, ppp0, etc.
   * -p (protocol) — the network protocol where your filtering process takes place. It can be either tcp, udp, udplite, icmp, sctp, icmpv6, and so on. Alternatively, you can type all to choose every protocol.
   * -s (source) — the address from which traffic comes from. You can add a hostname or IP address.
   * --dport (destination port) — the destination port number of a protocol, such as 22 (SSH), 443 (https), etc.
   * -j (target) — the target name (ACCEPT, DROP, RETURN). You need to insert this every time you make a new rule.

If you want to use all of them, you must write the command in this order:

```
sudo iptables -t <table name> -A <chain> -i <interface> -p <protocol (tcp/udp) > -s <source> --dport <port no.>  -j <target>
```

Once you understand the basic syntax, you can start configuring the firewall to give more security to your server. For this iptables tutorial, we are going to use the INPUT chain as an example.

And whenever you wanted to see the list of rules just use the command below:
```
sudo iptables -t <table name> -nvL #if you does not define table name it will default to filter table
```

<h2>Test case scenario:</h2>

<h2>How to achieve a soluiton:</h2>
The coming headers are nodes which you are assumed to have ssh access to them.

<h3>Client-1:</h3>
So here we need to accept ssh connection from anywhere so we achieve that through running the command below:

```
iptables -A INPUT -p tcp --dport 22 -j ACCEPT 
```

<h3>Router:</h3>

Set up IP FORWARDing and Masquerading in order to provide internet access to the specified hosts IP masquerading is a technique that hides an entire IP address space, usually consisting of private IP addresses, behind a single IP address in another, usually public address space. The hidden addresses are mapped to a single (public) IP address as the source address of the outgoing IP packets so they appear as originating not from the hidden host but from the routing device itself. Because of the popularity of this technique to conserve IPv4 address space, the term NAT has become virtually synonymous with IP masquerading. 
The term IP Forwarding describes sending a network package from one network interface to another one on the same device. It should be enabled when you want your system to act as a router that transfers IP packets from one network to another.
On a Linux system the Linux kernel has a variable named `ip_forward` that keeps this value. It is accessible using the file `/proc/sys/net/ipv4/ip_forward`. The default value is 0 which means no IP Forwarding, because a regular user who runs a single computer without further components is not in need of that, usually. In contrast, for routers, gateways and VPN servers it is quite an essential feature.

```
echo 1 > /proc/sys/net/ipv4/ip_forward
    
iptables --table nat --append POSTROUTING --out-interface eth0 -j MASQUERADE
iptables --append FORWARD --in-interface eth1 -j ACCEPT

iptables -I FORWARD -i eth0 -o eth1 -j ACCEPT
iptables -I FORWARD -i eth1 -o eth0 -j ACCEPT
```
  
Forwarding requests from port 8080 to the webserver service listening to port 80 on 	server-1 webserver:
	
```
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 -j DNAT --to 192.168.72.10:80 

iptables -t nat -A PREROUTING -i eth0 -p tcp -s 45.82.137.121 --dport 8080 -j DNAT --to 192.168.72.10:80

iptables -t nat -A PREROUTING -i eth0 -p tcp -s 37.152.183.225 --dport 8080 -j DNAT --to 192.168.72.10:80

iptables -t nat -A PREROUTING -i eth0 -p tcp -s 192.168.72.3 --dport 8080 -j DNAT --to 192.168.72.10:80


iptables -A FORWARD -p tcp -s 45.82.137.121 -d 192.168.72.10 --dport 80 -j ACCEPT 

iptables -A FORWARD -p tcp -s 37.152.183.225 -d 192.168.72.10 --dport 80 -j ACCEPT

iptables -A FORWARD -p tcp -s 192.168.72.3 -d 192.168.72.10 --dport 80 -j ACCEPT
```

Forwarding requests from port 21 to the ftp server listening to port 21 on 	server-1 webserver:
  
```
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 21 -j DNAT --to 192.168.72.10:21 >> changes the destination

iptables -t nat -A PREROUTING -i eth0 -p tcp -s 37.152.183.225  --dport 21 -j DNAT --to 192.168.72.10:21

iptables -t nat -A PREROUTING -i eth0 -p tcp -s 192.168.72.3  --dport 21 -j DNAT --to 192.168.72.10:21

iptables -t nat -A PREROUTING -i eth0 -p tcp -s 45.82.137.121  --dport 21 -j DNAT --to 192.168.72.10:21


iptables -A FORWARD -p tcp -d 192.168.72.10 --dport 21 -j ACCEPT 

iptables -A FORWARD -p tcp -s 45.82.137.121 -d 192.168.72.10 --dport 21 -j ACCEPT 

iptables -A FORWARD -p tcp -s 37.152.183.225 -d 192.168.72.10 --dport 21 -j ACCEPT 

iptables -A FORWARD -p tcp -s 192.168.72.3 -d 192.168.72.10 --dport 21 -j ACCEPT 
```

FTP uses 21TCP to establish a connection, but 20TCP to send/receive data:

Forwarding requests from port 20 to the ftp server listening to port 20 on 	server-1 webserver
```	
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 20 -j DNAT --to 192.168.72.10:20  

iptables -t nat -A PREROUTING -i eth0 -p tcp -s 37.152.183.225 --dport 20 -j DNAT --to 192.168.72.10:20

iptables -t nat -A PREROUTING -i eth0 -p tcp -s 192.168.72.3 --dport 20 -j DNAT --to 192.168.72.10:20

iptables -t nat -A PREROUTING -i eth0 -p tcp -s 45.82.137.121 --dport 20 -j DNAT --to 192.168.72.10:20

iptables -A FORWARD -p tcp -d 192.168.72.10 --dport 20 -j ACCEPT
```
	

Forwarding requests from port 60000-61000 to the ftp server listening to port 60000-61000 on server-1 webserver
  
```
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 60000:61000 -j DNAT --to 192.168.72.10:60000-61000
iptables -A FORWARD -p tcp -d 192.168.72.10 --dport 60000:61000 -j ACCEPT
```

Give hosts access to the http server:

```
iptables -A INPUT -p tcp -s 192.168.72.3 --dport 80 -j ACCEPT
iptables -A INPUT -p tcp -s 45.82.137.121 --dport 80 -j ACCEPT
iptables -A INPUT -p tcp -s 37.152.183.225 --dport 80 -j ACCEPT
```
Allow client-1 to have ssh access:

```
iptables -A INPUT -p tcp -s 37.152.183.225 --dport 22 -j ACCEPT
```


Finally all incoming and forwarding connections which we are the initaitor will come in:

```
iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

This will change the default policy to drop every incoming packets unless they are listed the rules:

```
sudo iptables -P INPUT DROP
```

	

<h3>Client2:</h3>
Change default gateway to our router ( traceroute can prove that )

```
route add default gw 192.168.72.6 eth0  
```

Give access to router for ssh and drop the rest:

```
iptables -A INPUT -p tcp -s 192.168.72.6 --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```
	
In order to change ttl we have to use mangle I prefered to increment 100 to default:
```
iptables -t mangle -A POSTROUTING -j TTL --tll-inc 100
```


<h3>Server:</h3>
change default gateway to our router ( traceroute can prove that )

```
route add default gw 192.168.72.6 eth0  
```


This guide uses the VSFTPD (VSFTPD stands for “Very Secure FTP Daemon software package”). It’s a relatively easy software utility to use for creating an FTP server. And the instructions assumes that centos7 is installed
Start by updating the package manager:
```
sudo yum update
```

Install VSFTPD software with the following command:
```
sudo yum install vsftpd
```

Start the service and set it to launch when the system boots with the following:
```
sudo systemctl start vsftpd
sudo systemctl enable vsftpd
```

Configuring VSFTPD:
Before starting, create a copy of the default configuration file:
```
sudo cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.default
```
Next, edit the configuration file with the following command:
```
sudo vi /etc/vsftpd/vsftpd.conf
```

Set your FTP server to disable anonymous users and allow local users.Find the following entries in the configuration file, and edit them to match the following:
```
anonymous_enable=NO
local_enable=YES
```
Next, allow a logged-in user to upload files to your FTP server.

Find the following entry, and edit to match as follows:
```
write_enable=YES
```

Limit FTP users to their own home directory. This is often called jail or chroot jail. Find and adjust the entry to match the following:
```
chroot_local_user=YES
allow_writeable_chroot=YES
```

The vsftpd utility provides a way to create an approved user list. To manage users this way, find the userlist_enable entry, then edit the file to look as follows:
```
userlist_enable=YES
userlist_file=/etc/vsftpd/user_list
userlist_deny=NO
```

You can now edit the /etc/vsftpd/user_list file, and add your list of users. (List one per line.) The userlist_deny option lets you specify users to be included; setting it to yes would change the list to users that are blocked.

Once you’re finished editing the configuration file, save your changes. Restart the vsftpd service to apply changes:
```
sudo systemctl restart vsftpd
```
FTP passive & active mode :
By default (active mode), data connections are established from the sender to the receiver. For the output of ls, the data is sent by the server, so the server attempts to open a connection to the client. This worked well when FTP was invented, but nowadays, clients are often behind a firewall or NAT which may or may not support active FTP. Switch to passive mode, where the client always initiates the data connection.

ftp server configs :
passive connections port range in order to make it easier to filter them
```
pasv_min_port=60000
pasv_max_port=61000
```
  

Create a New FTP User:
To create a new FTP user enter the following:
```
sudo adduser testuser
sudo passwd testuser
```
Add the new user to the userlist:
```
echo “testuser” | sudo tee –a /etc/vsftpd/user_list
```
Create a directory for the new user, and adjust permissions:
```
sudo mkdir –p /home/testuser/ftp/upload
sudo chmod 550 /home/testuser/ftp
sudo chmod 750 /home/testuser/ftp/upload
sudo chown –R testuser: /home/testuser/ftp
```
This creates a home/testuser directory for the new user, with a special directory for uploads. It sets permissions for uploads only to the /uploads directory.
Now, you can log in to your FTP server with the user you created:
```
ftp 192.168.72.10 # Replace this IP address with the one from your system. 
```

Install NGINX webserver 
Install Extra Packages for Enterprise Linux (EPEL)
To install EPEL, run the following command using the Yum package manager:
```
sudo yum install -y epel-release
```
Install Nginx:
```
sudo yum –y install nginx
```
Start Nginx Service:
```
sudo systemctl start nginx
sudo systemctl status nginx
```
Configure Nginx to Start on Boot:
```
sudo systemctl enable nginx
```
You can check Nginx to see if it is running on port 80 by executing the following command:
```
curl localhost
```

Give access to router for ssh and drop the rest
```
iptables -A INPUT -p tcp -s 192.168.72.6 --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```








