# Communication-between-two-containers-using-VxLan-overlay-network-and-openvswitch
Environment setup: At first you have to take two virtual machine and virtual machine should ping one another. I am using centos9 for this demo and I have taken two Centos9 virtual machines from ec2 instances. For Ec2 security group I am allowing all traffic to cope up with connectivity problem. Two virtual machine should be in same vpc so that you can ping one from another.Here is the security group picture

<img width="713" alt="security group" src="https://github.com/nobelrakib/Communication-between-two-containers-using-VxLan-overlay-network-and-openvswitch/assets/53372696/ec9d4a01-c7d9-45f0-8ec8-1b8ccf276b78">

What we are going to make

<img width="470" alt="Screenshot 2023-07-18 233955" src="https://github.com/nobelrakib/Communication-between-two-containers-using-VxLan-overlay-network-and-openvswitch/assets/53372696/4f8ca7d9-da44-4efc-b83e-36709feff2c5">
