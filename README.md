# Creating linux namespace and communicate via Linux bridge 


In this article we will see how to communicate with outside network from isolated Linux namespace. In addition, we will cover:

1. How to create linux namespace
2. Configure IP addresses for virtual ethernet pair
3. Inter network communication between two namespace 
4. Communication between namespace and linux bridge 
5. Communication between namespace and root namespace 
6. We will see how to communicate with outside network

# How to create Linux namespace
Namespace is a technology of Linux Kernel which used to isolated resources from outer world. That means, A set of processes can’t see from the another namespaces. To create namespace-
```
sudo ip netns add red 
sudo ip netns add green
```
To view the list of namespaces
```
sudo ip netnes list
```
Output:
```
$ sudo ip netns list
```
green
red

To view existing interfaces in root namespace 
```
$ ifconfig
```
Output:
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 9001
        inet 172.31.0.97  netmask 255.255.240.0  broadcast 172.31.15.255
        inet6 fe80::898:55ff:fe35:dd88  prefixlen 64  scopeid 0x20<link>
        ether 0a:98:55:35:dd:88  txqueuelen 1000  (Ethernet)
        RX packets 838  bytes 323478 (323.4 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 884  bytes 107277 (107.2 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 198  bytes 17190 (17.1 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 198  bytes 17190 (17.1 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

From above we can see there are two interfaces exist one is eth0 and another is loopback. Now we will create interfaces for red and green namespaces
```
$sudo ip link add veth0 type veth peer name ceth0 
```
Now, if we type :
```
$ sudo ip link
```

Output:
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 0a:98:55:35:dd:88 brd ff:ff:ff:ff:ff:ff
3: ceth0@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 3e:a3:63:a0:22:87 brd ff:ff:ff:ff:ff:ff
4: veth0@ceth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 82:c5:75:60:23:00 brd ff:ff:ff:ff:ff:ff

We can see above veth0 and ceth0 has been created but they are currently down. Now, we have to make those up. To do this, we will create a Linux bridge which helps to establish inter communication between two namespaces. 
```
$ sudo ip link add br0 type bridge 
```
To check whether the network bridge has been created or not type:
```
$ sudo ip link
```
  
Output:
  
  
<img width="560" alt="linkloopback 000000000000 brd 00000000000" src="https://user-images.githubusercontent.com/20005885/149611202-adb80ef9-1c87-4ca2-aa02-02f6b3b97646.png">


So the network bridge has been created!!!Now we will make an interface for the bridge.
```
$ sudo ip link set br0 up
```
This will make state up 
```
$ sudo ip add add 192.168.0.1/16 dev br0
$ sudo ip addr
```
Output:

<img width="568" alt="noqueue" src="https://user-images.githubusercontent.com/20005885/149611417-70119c74-c63f-49a2-b3c0-42d00f9b0b09.png">


The interface for br0 has been created with ip: 192.168.0.1/16.

Now we will create interface for red namespace. So 
```
$ sudo ip addr add 192.168.0.2/16 dev veth0
$ sudo ip addr
 ```
<img width="568" alt="noop state" src="https://user-images.githubusercontent.com/20005885/149611423-e4469031-e3be-4403-97fb-d4441ecff46b.png">

ip 192.168.0.2/16 has been created for veth0 interface. To make this up
```
$ sudo ip link set veth0 up
```
We will enter red namespace to set an IP address:
```
$ sudo ip netns exec red sh
```
See, we have only loopback interface in red namespace. We have to connect ceth0.
```
$ exit
```
```
$ sudo ip link set ceth0 netns red
$ sudo ip netns exec red sh
# ip addr add 192.168.0.3/16 dev ceth0
# ip addr
```
Output:
  
<img width="568" alt="qlen 1000" src="https://user-images.githubusercontent.com/20005885/149611448-17ffa224-e0e3-42d6-8348-e5f2aaf180aa.png">

Make state up 
```
# ip link set ceth0 up
# ip addr
```
<img width="568" alt="Screen Shot 2022-01-10 at 8 13 36 PM" src="https://user-images.githubusercontent.com/20005885/149611460-3f2959b1-97e7-499b-b9e5-7975d025d9c6.png">

After making the state up we can see “OVERLAYDOWN”. What is the reason? The answer is the other side vets pair veth0 is still down. So we should make this up to solve the issue. To do this,
```
# exit
```
```
$ sudo ip link set veth0 up
```
Now we will ping to br0 and check if we will get any response 
```
$ sudo ip link set ceth0 netns red
# ping 192.168.0.1
```

Output:
  
<img width="568" alt="ping 192 168 0" src="https://user-images.githubusercontent.com/20005885/149611468-98d8627c-f00b-410e-95ef-381d44dcd2e6.png">

We are not getting any response from br0. Let’s check br0 state
```
# exit
```
```
$ sudo ip addr
```
<img width="568" alt="roup default qlen 1000" src="https://user-images.githubusercontent.com/20005885/149611473-1901b10b-0e12-4b7c-8cfa-3c5e6b0e2ab6.png">

We can see the state is unknown. So we have to connect veth0 with bridge network 
```
$ sudo ip link set veth0 master br0
$ sudo ip addr
```
Output:
  
<img width="568" alt="Screen Shot 2022-01-10 at 8 54 42 PM" src="https://user-images.githubusercontent.com/20005885/149611642-1108ba5a-8f57-4f30-94d2-5edcd1e31dfa.png">

We can see bro state up. Now we will check if we get any response from red namespace.
```
$ sudo ip netns exec red sh
```
```
# ping 192.168.0.1
```
<img width="568" alt="ping 192 168 0" src="https://user-images.githubusercontent.com/20005885/149611499-cfe4bdad-0d6b-44b8-9de2-640547fe629c.png">
  
 We are seeing that, we are getting ICMP(Internet Control Message Protocol) response. 

Likewise, We will setup ip address for the green namespaces. Later, we will check if we get response from two isolated namespaces and vise Versa 

Now, We will try to ping from red namespace to green namespace.
```
$ sudo ip netns exec red sh
```
<img width="568" alt="PING 192 168 0 4 (192 168 0 4) 56(84) bytes of data" src="https://user-images.githubusercontent.com/20005885/149611500-9c22e9da-6187-49bd-886a-ade1e29bf666.png">

We are getting response from green network namespace.
  
  
  
![192 168 0 5](https://user-images.githubusercontent.com/20005885/149611555-0f040d78-e5f9-4aa2-9504-225759414c64.png)

  
  
  
  
The diagram illustrate how to traverse packets.

Now, we are going to ping to eth0 which is root namespace. 
  
<img width="568" alt="sudo ip netns" src="https://user-images.githubusercontent.com/20005885/149611840-76009f7f-14f6-4af9-9ef0-e7576a86e26e.png">

  
We could not communicate with tooth namespace. As eth0 is from different network. We should have a default gateway to forward any request which is not matched any of the IP addresses in ip table.
```
$ route
```
Output: 

<img width="568" alt="Kernel IP routing table" src="https://user-images.githubusercontent.com/20005885/149611853-d60dc4bf-eaef-45eb-a476-10ddd06d97f9.png">

So, we can see that green namespace does not have any default gateway. We will add a default gateway in route table:
```
$ ip route add default via 192.168.0.1
$ route
```
Output: 

<img width="568" alt="Kernel IP routing table" src="https://user-images.githubusercontent.com/20005885/149611867-e03c13ff-be99-4b44-b8b7-40e9af11a0e7.png">

We can see default gateway has been created. Let’s ping to eth0.
```
$ ping 172.31.0.97 
```
  
<img width="568" alt="PING 172 31 0 5 (172 31 0 97) 56(84) bytes of data" src="https://user-images.githubusercontent.com/20005885/149611882-52fb6b2b-3b63-414a-9adc-68c68d40e168.png">

We are receiving packets from eth0 interface.

# Communicate with outside network 8.8.8.8 from green namespace:
```
# ping 8.8.8.8
```
output: 

<img width="568" alt="ping 173 194 36 16" src="https://user-images.githubusercontent.com/20005885/149611912-a91e04e3-7fea-4606-a980-0667bf65f3d5.png">

Green namespace could not communicate with outside network through dns server 8.8.8.8. First check whether ip forwarding status is enabled 
```
$ cat /proc/sys/net/ipv4/ip_forward
```
Output: 
0 

That means it’s not enable. So we need to make this enable.
```
$ sudo nano /proc/sys/net/ipv4/ip_forward
$ cat /proc/sys/net/ipv4/ip_forward
```
Output: 
1
Ip forwarding status has been enabled 
 To communicate with outside network from isolated namespace like green. We should design an ip table such a way that private IP address will convert into public IP address. To do so, we have to apply Masquerade rule in ip table to communicate with external IP address. This that case Masquerade rule allows:<br>
**- When a request create from a private IP address , Masquerade rule allows the private IP address convert public ip and transmit the request into internet with and external IP address.**<br>
**- When the destination host receives the request , it believes that the request it receives from that external IP address. **<br>
**- After that whenever, Masquerade table receives packets, it looks into ip table to ensure whether the IP address belongs to its table.**<br>
**- Finally, it converts the external ip into private ip which used fro packet forwarding.**

To use Masquerade rule: 
```
$ sudo iptables \
> -t nat \
> -A POSTROUTING \
> -s 192.168.0.0/16 \
> -j MASQUERADE
```
Here,
**-t initiate the table command** 
**-A specify that we are appending  POSTROUTING rule** 
**-s specifies the source address** 
**-j marks which action is being performed**

Now, if we ping to 8.8.8.8 from green namespace 
```
$ sudo ip  netns exec green sh
```
```
# ping 8.8.8.8
```
<img width="568" alt="PING 8 8 8 8 (8 8 8 8) 56(84) bytes of data" src="https://user-images.githubusercontent.com/20005885/149611931-631e772d-337d-4618-91cf-5ecbc7cf57bb.png">

Finally, we are receiving imp response from 8.8.8.8!!!

In this section, we will try to establish communication between two private ip of different server.

We have succesfully establish communication with ouside network. We are going to ping to a prive ip address of different network. 


