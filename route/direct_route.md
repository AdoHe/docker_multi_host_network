We have seen methods where the containers have used the IP address of host to communicate with each other, but there are solutions to inter-connect the containers by using their own IPs. If we are using the containers own IPs for routing then it is important that we shall be able to distinguish based on IP which container is running on Host1 and which one is running on on Host2. Steps below illustrate how to use static route to enable multi-host docker network communication.

## Prerequistie
* two VMs(Host1 and Host2) with the same subnet
* docker 1.6 installed on each host

## Reconfigure docker0 bridge
Since we route traffic based on containers' IPs, we have to avoid IP address conflicts, so we will allocate different subnets for Host1 and Host2.
On Host1, run this commands:

```
[root@Host1 ~]# ifconfig docker0 172.17.128.254/18
[root@Host1 ~]# service docker restart
```
run the same command on Host2:

```
[root@Host2 ~]# ifconfig docker0 172.17.192.254/18
[root@Host2 ~]# service docker restart
```
After this, `docker daemon` will use the subnets configured to allocate IP for each new container.

## Add route and reset iptables
On Host1, we need to forward packets which destination is `172.17.192.0/18` to Host2, also we have to reset iptables rules. Run these commands on Host1:

```
[root@Host1 ~]# route add -net 172.17.192.0 netmask 255.255.192.0 gw 10.198.194.217
[root@Host1 ~]# iptables -t nat -F POSTROUTING
[root@Host1 ~]# iptables -t nat -A POSTROUTING -s 172.17.128.0/18 ! -d 172.17.0.0/16 -j MASQUERADE
```
On Host2, run the opposite commands:

```
[root@Host1 ~]# route add -net 172.17.128.0 netmask 255.255.192.0 gw 10.198.194.216
[root@Host1 ~]# iptables -t nat -F POSTROUTING
[root@Host1 ~]# iptables -t nat -A POSTROUTING -s 172.17.192.0/18 ! -d 172.17.0.0/16 -j MASQUERADE
```
By default, `docker` sets up a rule to the nat table to masquerade all IP packets that are leaving the machine. In this test case, we definitely don’t want this, therefore we delete all MASQUERADE rules with -F option. At this point we already would be able to make the connection from one container to other and vice verse, but the containers would not be able to communicate with the outside world, therefore an iptables rule needs to be added to masquerade the packets that are going outside of 172.17.0.0/16.

## Test with qperf
On Host1, start `qperf_server` container:

```
[root@Host2 ~]# docker run -ti --rm --name qperf_server -p 4000:4000 -p 4001:4001 qperf /bin/bash
[root@1c70fed0cc70 /]# qperf -lp 4000
```

On Host2, start `qperf_client` container and run the the test script:

```
[root@Host2 ~]# docker run -ti --rm --name qperf_client qperf /bin/bash
[root@a55793ee24f5 /]# ./qperf_test.sh
```

Such kind of direct routing from one vm to other vm works great and easy to set up, but cannot be used if the hosts are not on the same subnet.