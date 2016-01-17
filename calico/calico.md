# Calico

Compared with other mutli host networking approaches, calico represents a new approach to virtual networking, which based on the same scalable IP networking principles as the Internet. More specificly, calico does not use traditional overlay, instead providing a pure Layer 3 approach to data center networking. Calico is simple to deploy and diagnose, provides a rich security policy, supports both IPv4 and IPv6 and can be used across a combination of bare-metal, VM and container workloads. Steps below illustrate how to use Calico to enable multi-host docker network communication.



## Prerequisite
* Two VMs(Host1 and Host2)
* Docker 1.8 installed on each host
* calicoctl installed on each host
* Accessible Etcd cluster(Here we run single-node Etcd cluster on Host1)


## Start Etcd
start Etcd on Host1:
```
[root@Host1 etcd-v2.2.1-linux-amd64]# ./etcd -listen-peer-urls "http://<Host1_IP>:2380,http://<Host1_IP>:7001" -listen-client-urls "http://<Host1_IP>:2379,http://<Host1_IP>:4001" -advertise-client-urls "http://<Host1_IP>:2379,http://<Host1_IP>:4001" &
```

## Start Calico
on Host1 run `calico-node` container:
```
[root@Host1 ~]# export ETCD_AUTHORITY=<Host1_IP>:2379
[root@Host1 ~]# ./calicoctl node --ip=<Host1_IP>
```
run the same command on Host2:
```
[root@Host2 ~]# export ETCD_AUTHORITY=<Host1_IP>:2379
[root@Host2 ~]# ./calicoctl node --ip=<Host2_IP>
```

## Create Profile
on Host1 create a `test` profile:
```
[root@Host2 ~]# ./calicoctl profile add test
```

## Start Containers
on Host1, start the `qperf_server` container:
```
[root@Host1 ~]# export DOCKER_HOST=localhost:2377
[root@Host1 ~]# docker run --name qperf_server -e CALICO_IP=auto -e CALICO_PROFILE=test -p 4000:4000 -p 4002:4002 -td qperf
```
on Host2, start the `qperf_client` container:
```
[root@Host2 ~]# export DOCKER_HOST=localhost:2377
[root@Host2 ~]# docker run --name qperf_client -e CALICO_IP=auto -e CALICO_PROFILE=test -td qperf
```

## Test with qperf
on Host1, start `qperf server` in container:
```
[root@Host1 ~]# docker exec -it qperf_server /bin/bash
[root@101da1cf73d1 /]# qperf -lp 4000
```
on Host2, start `qperf client` in container:
```
[root@Host2 ~]# docker exec -it qperf_client /bin/bash
[root@155fde394ac2 /]# qperf -lp 4000 -ip 4002 192.168.0.1 tcp_bw tcp_lat
```

This amazing project provides direct connectivity through IP routing, fine-grained policy rules and so on by using `iptables` and kernel `route table`. Since everything is based on IP, so if your hosts are within different subnets, you have to be careful to make sure the routers on your network knows how to forward Calico packets. One possible solution is [IPIP](https://en.wikipedia.org/wiki/IP_in_IP), you can reference <http://www.projectcalico.org> for more detail. 