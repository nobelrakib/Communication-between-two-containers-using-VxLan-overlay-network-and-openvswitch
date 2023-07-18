# Communication-between-two-containers-using-VxLan-overlay-network-and-openvswitch
Environment setup: At first you have to take two virtual machine and virtual machine should ping one another. I am using centos9 for this demo and I have taken two Centos9 virtual machines from ec2 instances. For Ec2 security group I am allowing all traffic to cope up with connectivity problem. Two virtual machine should be in same vpc so that you can ping one from another.Here is the security group picture

<img width="713" alt="security group" src="https://github.com/nobelrakib/Communication-between-two-containers-using-VxLan-overlay-network-and-openvswitch/assets/53372696/ec9d4a01-c7d9-45f0-8ec8-1b8ccf276b78">

What we are going to make

<img width="470" alt="Screenshot 2023-07-18 233955" src="https://github.com/nobelrakib/Communication-between-two-containers-using-VxLan-overlay-network-and-openvswitch/assets/53372696/4f8ca7d9-da44-4efc-b83e-36709feff2c5">

What is vxlan?

In modern network topologies, virtualized Layer 2 networks are built on top of the underlying physical infrastructure known as the underlay network using overlay networks like VXLAN. VXLAN uses the VNI, a 24-bit identifier, to distinguish between distinct virtual networks within the overlay. VTEPs act as gateways between the overlay and underlay networks by encapsulating and decapsulating the VXLAN packets. Overlay networks using VXLAN, VNI, and VTEP technologies make it possible for flexible provisioning, multi-tenancy, and isolation while also enabling flexible network topologies that are scalable and adaptable in virtualized and cloud settings.

Let’s create a overlay network with openvswitch which will connect two docker container in two diffrent virtual machine.

For virtual machine one follow these steps

```
#log in using key
1.ssh -i vxlan.pem ubuntu@ec2-13-231-150-149.ap-northeast-1.compute.amazonaws.com
#install docker,openvswitch and necessary dependencies
2.sudo apt update
3.sudo apt -y install net-tools docker.io openvswitch-switch
#now creat a bridge using openvswitch
4.sudo ovs-vsctl add-br ovs-br0
#add an interface with this bridge 
5.sudo ovs-vsctl add-port ovs-br0 veth0 -- set interface veth0 type=internal
#now assign an ip to this bridge
6.sudo ip address add 192.168.1.1/24 dev veth0
#initially this interface will down need to up it
7.sudo ip link set dev veth0 up
#now we will create a docker file using vim
#check the interface of root namespace
root@ip-172-31-15-205:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc fq_codel state UP group default qlen 1000
    link/ether 0a:00:06:7d:19:4d brd ff:ff:ff:ff:ff:ff
    inet 172.31.15.205/20 metric 100 brd 172.31.15.255 scope global dynamic eth0
       valid_lft 3246sec preferred_lft 3246sec
    inet6 fe80::800:6ff:fe7d:194d/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:5f:57:e8:32 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:5fff:fe57:e832/64 scope link
       valid_lft forever preferred_lft forever
4: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 1a:2a:64:50:da:83 brd ff:ff:ff:ff:ff:ff
5: ovs-br0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether e2:7b:85:62:5b:44 brd ff:ff:ff:ff:ff:ff
6: veth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether da:40:14:07:02:77 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.1/24 scope global veth0
       valid_lft forever preferred_lft forever
    inet6 fe80::d840:14ff:fe07:277/64 scope link
       valid_lft forever preferred_lft forever
16: d43cf53faa994_l@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UP group default qlen 1000
    link/ether be:29:5c:6a:88:d5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::f47d:9ff:fe72:7eff/64 scope link
       valid_lft forever preferred_lft forever
17: vxlan_sys_4789: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65000 qdisc noqueue master ovs-system state UNKNOWN group default qlen 1000
    link/ether 76:53:39:51:15:f0 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::e0dd:c4ff:fe3c:ec6d/64 scope link
       valid_lft forever preferred_lft forever

8.Vim Dockerfile
FROM ubuntu

RUN apt update
RUN apt install -y net-tools
RUN apt install -y iputils-ping
RUN apt install -y iproute2

CMD ["sleep", "30000"]

#now build the image and create container with no network
9.sudo docker run -d --net=none --name docker1 ubuntu-docker

#now this docker container has no ip as it is not connected with any 
#network assign an ip address to it target its gateway to our bridge
10.sudo ovs-docker add-port ovs-br0 eth0 docker1 --ipaddress=192.168.1.11/24 --gateway=192.168.1.1
#now connect to vxlan with our bridge created by openvswith and give it a key
#which will same for two server and assign remote server with which 
#it will be connected
11.sudo ovs-vsctl add-port ovs-br0 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=172.31.2.2 options:key=1000
```

set up for server 1 is done now do the same thing for server 2

```
#log in using key
1.ssh -i "vxlan.pem" ubuntu@ec2-18-182-25-103.ap-northeast-1.compute.amazonaws.com
#install docker,openvswitch and necessary dependencies
2.sudo apt update
3.sudo apt -y install net-tools docker.io openvswitch-switch
#now creat a bridge using openvswitch
4.sudo ovs-vsctl add-br ovs-br0
#add an interface with this bridge 
5.sudo ovs-vsctl add-port ovs-br0 veth0 -- set interface veth0 type=internal
#now assign an ip to this bridge
6.sudo ip address add 192.168.1.1/24 dev veth0
#initially this interface will down need to up it
7.sudo ip link set dev veth0 up
#now we will create a docker file using vim
#check the interface of root namespace
8.Vim Dockerfile
FROM ubuntu

RUN apt update
RUN apt install -y net-tools
RUN apt install -y iputils-ping
RUN apt install -y iproute2

CMD ["sleep", "30000"]

root@ip-172-31-2-2:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc fq_codel state UP group default qlen 1000
    link/ether 0a:7e:dd:b0:15:a7 brd ff:ff:ff:ff:ff:ff
    inet 172.31.2.2/20 metric 100 brd 172.31.15.255 scope global dynamic eth0
       valid_lft 2957sec preferred_lft 2957sec
    inet6 fe80::87e:ddff:feb0:15a7/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:81:99:1c:5d brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:81ff:fe99:1c5d/64 scope link
       valid_lft forever preferred_lft forever
4: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 0a:0b:2d:41:2c:14 brd ff:ff:ff:ff:ff:ff
5: ovs-br0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether fe:40:2f:0f:87:4c brd ff:ff:ff:ff:ff:ff
6: veth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 6a:2e:5d:0e:5a:72 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.1/24 scope global veth0
       valid_lft forever preferred_lft forever
    inet6 fe80::682e:5dff:fe0e:5a72/64 scope link
       valid_lft forever preferred_lft forever
16: b09975a64e704_l@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UP group default qlen 1000
    link/ether 3a:46:e7:f3:5e:e5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::20f8:d0ff:fe85:5244/64 scope link
       valid_lft forever preferred_lft forever
17: vxlan_sys_4789: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65000 qdisc noqueue master ovs-system state UNKNOWN group default qlen 1000
    link/ether 7e:3b:c9:5c:61:7f brd ff:ff:ff:ff:ff:ff
    inet6 fe80::c17:d4ff:fef4:e8d9/64 scope link
       valid_lft forever preferred_lft forever

#now build the image and create container with no network
9.sudo docker run -d --net=none --name docker2 ubuntu-docker

#now this docker container has no ip as it is not connected with any 
#network assign an ip address to it target its gateway to our bridge
10.sudo ovs-docker add-port ovs-br0 eth0 docker2 --ipaddress=192.168.1.12/24 --gateway=192.168.1.1
#now connect to vxlan with our bridge created by openvswith and give it a key
#which will same for two server and assign remote server with which 
#it will be connected
11.sudo ovs-vsctl add-port ovs-br0 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=172.31.15.205  options:key=1000
```

set up for server 2 is also done now lets ping from one machine to another
#machine container and see the output

Lets try to ping second machine container named docker2(ip :192.168.1.12)from first machine container docker1(ip :192.168.1.11)

<img width="769" alt="Screenshot 2023-07-18 233519" src="https://github.com/nobelrakib/Communication-between-two-containers-using-VxLan-overlay-network-and-openvswitch/assets/53372696/97cfe822-5860-48dd-a7d7-3fe35b5828da">

We are getting reply.

Now do the reverse.

<img width="746" alt="Screenshot 2023-07-18 233449" src="https://github.com/nobelrakib/Communication-between-two-containers-using-VxLan-overlay-network-and-openvswitch/assets/53372696/c6da49e1-f1bf-439b-8e10-b79e82b32629">

Here we also getting reply. So our tunneling with vxlan and openvswith is working fine for our two ec2 machines.


