




# [Use Kubeadm to install a basic cluster](#ref2)

이번 글을 `kubeadm`을 활용하여 K8s Cluster를 구축하는 방법에 대해서 다룰 것이다.

`kubeadm`은 다음과 같은 기능을 한다.
- Cluster Set Up
- Clsuter Lifecycle Functions: Bootstrp Tokens, Cluster Upgrade

그리고 다음과 같이 활용할 수 있다.
- Kubernetes Try Out
- Cluster Setup 자동화, Application Test

또한 노트북, 클라우드 서버, 라즈베리 파이등 다양한 환경에서 사용가능하다.


### Objectives

- Single Control-Plane Kubernetes Cluster 설치하기
- Pod Network 설치하기

### Installing Kubeadm on Your Host

[install-kubeadm](#ref3)

#### Requirement
- Linux Host
- 2GB or More RAM per machine
- 2CPUS or More
- Full Netrwork Connectivity between All Mahcines
- Unique Hostname, MAC Address, Product_uuid for Every Node
- Certain Port Open
- Swap Disabled: Kubelet이 원활히 작동하기 위해서 필요

**MAC Address란**

컴퓨터간 데이터를 전송하기 위해있는 컴퓨터의 물리적 주소.

"IP 주소간 통신은 각 라우터 hop에서 일어나는 MAC 주소와 MAC주소 통신의 연속적인 과정이다."

![](https://user-images.githubusercontent.com/24274424/86514723-94b77900-be4e-11ea-8456-ad39b27d9ba9.png)

다음과 같은 명령어를 통해서 MAC address를 파악할 수 있다.

```
ifconfig -a
```

**prdocut_uuid**

메인보드 제작자가 BIOS DMI 정보를 인코딩 한 것. 
메인보드를 식별하는데 사용될 수 있다.


#### Check Network Adapeters

만약 두 개이상의 Network Adpater를 보유하고 있고 Default Route로 서로 통신할 수 없는 경우,
IP routes를 추가하여 Kubernetes Cluster Address가 적절한 Adatper로 갈 수 있도록 한다.

- [Routing Table](#ref5)

라우터 혹은 네트워크 호스트에 저장되어 있는 데이터테이블.
물리적 네트워크 주소에 대한 라우트 리스트를 가지고 있으며 거리(매트릭)을 포함하고 있는 경우도 있다.


```
# 라우팅테이블: Gateway를 거쳐야 Destination을 거칠 수 있다.
# default: 어떤 라우팅도 적용하지 않았을 때의 목적지
# Destination도 세가지 타입이 있다. default, individual hosts, subnet
# Gateway는 세가지 타입이 있다. individual hosts, interface, MAC address
$ netstat -rn

Internet:
Destination        Gateway            Flags        Netif Expire
default            192.168.0.1        UGScg          en0       
127                127.0.0.1          UCS            lo0       
127.0.0.1          127.0.0.1          UH             lo0       
169.254            link#6             UCS            en0      !
192.168.0          link#6             UCS            en0      !
192.168.0.1/32     link#6             UCS            en0      !
192.168.0.1        90:9f:33:dd:5e:2   UHLWIir        en0   1173
192.168.0.6        a2:16:9c:b7:fa:63  UHLWIi         en0    776
192.168.0.7        ea:b6:4:6a:65:c0   UHLWIi         en0    625
192.168.0.9        7a:da:6d:5e:7a:6   UHLWIi         en0   1080
192.168.0.10/32    link#6             UCS            en0      !
224.0.0/4          link#6             UmCS           en0      !
224.0.0.251        1:0:5e:0:0:fb      UHmLWI         en0       
255.255.255.255/32 link#6             UCS            en0      !
```

- [Add Route on Linux](#ref4)


#### Letting iptables see bridged traffic

- [bridge](#ref6)

Linux `bridge`는 네트워크 스위치와 유사한 역할을 한다.
`bridge`는 연결된 인터페이스에 패킷을 포워딩한다.
주로 라우터 혹은 게이트웨이, VM 네트워크 네임스페이스에 패킷을 포워딩한다.

![](https://developers.redhat.com/blog/wp-content/uploads/2018/10/bridge.png)

- `br_netfilter`가 있어야함.
- `net.bridge.bridge-nf-call-iptables`을 1로 세팅

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

#### Check required ports

[Required Port](#ref7)가 모두 열려있어야한다.

```
# 해당명령어로 확인
telnet 127.0.0.1 6443
```

#### Installing runtime

**Container Runtime**

컨테이너를 활용하기 위해 Container Runtime을 활용하여 Linux Kernel을 활용한다.
주로 Namespace, C-Group, SELinux를 활용한다.
- Namespace: 프로세스 격리
- cgroup: 네임스페이스에 대한 관리 추적을 위한 역할

쿠버네티스를 사용하기 위해서는 모든 노드에 Container Runtime이 설치되어 있어야한다.
이를 활용하여 Pod가 생성된다.

```
# 리스트
Runtime	Path to Unix domain socket
Docker	/var/run/dockershim.sock
containerd	/run/containerd/containerd.sock
CRI-O	/var/run/crio/crio.sock
```
`systemd`는 init system에 의해서 시작되며 cgroup mangers의 역할을 한다.

참고로 Container Runtime과 Kubelet모두 `cgroupfs`를 사용하도록 설정할 수 있다.
그러나 이것은 결국 `systemd`와 `cgroupfs`가 동시에 cgroup manager의 역할을 하는 것이다.
이는 불안정적인 세팅이므로 Container Runtime과 Kubelet모두 `systemd`를 사용하도록 설정하는 것을 권장한다.


#### Configuring a cgroup driver

Container Runtime과 Kubelet모두 cgroup driver가 존재한다.

cgroups(Control Groups)는 프로세스들의 자원의 사용(CPU, 메모리, 디스크입출력, 네트워크 등)을 제한하고 격리시키는 리눅스 커널기능이다.

kubeadm의 `cgroupDriver`를 systemd로 설정할 것을 권장한다.
kubeadm은 kubelet을 systemd service와 유사하게 관리한다.

```
# kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.21.0
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```


[**systemd**](#ref9)

systemd is a suite of basic building blocks for a Linux system. It provides a system and service manager that runs as PID 1 and starts the rest of the system.





## Reference

<a name="#ref1" href="https://kubernetes.io/docs/reference/access-authn-authz/rbac/"> [1] Using RBAC Authorization [websites] (2022, Feb, 13)</a>

<a name="#ref2" href="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/"> [2] Creating a cluster with kubeadm [websites] (2022, Feb, 21)</a>

<a name="#ref3" href="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/"> [3] Installing kubeadm [websites] (2022, Feb, 21)</a>

<a name="#ref4" href="https://devconnected.com/how-to-add-route-on-linux/#:~:text=The%20easiest%20way%20to%20add,be%20used%20for%20this%20route.&text=By%20default%2C%20if%20you%20don,loopback%20excluded%2C%20will%20be%20selected."> [4] Add Route on Linux[websites] (2022, Feb, 27)</a>

<a name="#ref5" href="https://en.wikipedia.org/wiki/Routing_table."> [5] Routing Table[websites] (2022, Feb, 27)</a>

<a name="#ref6" href="https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking#bridge"> [6] Bridge [websites] (2022, Feb, 27)</a>

<a name="#ref7" href="https://kubernetes.io/docs/reference/ports-and-protocols/#control-plane"> [7] Ports and Protocols [websites] (2022, Feb, 27)</a>

<a name="#ref7" href="https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-drivers"> [8] Cgroup drivers [websites] (2022, Feb, 27)</a>

<a name="#ref8" href="https://systemd.io/"> [9] systemd [websites] (2022, Feb, 27)</a>

