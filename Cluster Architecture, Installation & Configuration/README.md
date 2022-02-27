# Cluster Architecture, Installation & Configuration: 25%

## Manage role Based access control(RBAC)

Role-Based Access Control(RBAC)는 Computer나 Network Resource의 접근을 각 User의 Role을 기반으로 규제하는 방법이다.

RBAC는 `rbac.authorization.k8s.io` API Group를 사용하여 Authorization 결정을 한다.
RBAC는 Kubernetes API를 통해서 동적으로 설정할 수 있다.

RBAC를 사용하기 위해서는 API Server를 다음과 같은 명령어를 통해서 시작해야한다.
- `--authorization-mode=RBAC`

```
# Example
kube-apiserver --authorization-mode=Example,RBAC --other-options --more-options
```

### API Object

RBAC API는 다음과 같은 Kubernetes object를 정의한다.
- Role
- ClusterRole
- RoleBinding
- ClusterRoleBinding

### Role and ClusterRole

Role과 ClusterRole은 허가를 추가하는 방식이며 거부에 대한 내용은 없다.
Role은 항상 특정 네임스페이스에서 작동한다.
따라서 Role 생성시에는 네임스페이스를 특정해야한다.

반면에 ClusterRole같은 경우에는 네임스페이스와 무관한 리소스이다.
Kubernetes의 Object의 경우 네임스페이스과 관련이 있는 Obejct도 있지만 관련이 없는 Object도 있기때문에 Role과 ClusterRole이 각각 존재한다.

보통 ClusterRole같은 경우에는 다음과 같이 사용한다.
1. 네임스페이스 리소스(예 pod)의 Permission을 정의하고 각 네임스페이스내에서 정의
2. 네임스페이스 리소스(예 pod)의 Permission을 정의하고 모든 네임스페이스내에서 정의
3. Cluster-Scope의 리소스의 Permission을 정의

네임스페이스안에서 Role을 정의한다면 Role을 사용하고 Cluster-Wide하게 Role을 정의한다면 ClusterRole을 사용한다.


### Role Example

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

`default` 네임스페이스안에 있는 `Pod`에 대하여 `["get", "watch", "list"]` 권한을 설정한다.

### ClusterRole Example

Cluster-Scope의 범위로 Permission을 설정하기 때문에 다음과 같이 활용할 수 있다.

- Cluster-Scoped Resource: Nodes
- Non-Resource Endpoints: `/healthz`
- 모든 네임스페이스에 대해 Namespaced Resource(Pods)에 권한을 부여할 수 있음

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Secret
  # objects is "secrets"
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

### RoleBinding and ClusterRoleBinding

RoleBinding은 User혹은 User집합에 Role에서 설정한 Permission을 부여하는 방법이다. (User, Group, ServiceAccount)
Role과 마찬가지로 RoleBinding도 네임스페이스내에서 정의된다.
반면에 ClusterRoleBinding은 Cluster-Wide한 권한을 부여한다.

RoleBinding의 경우 다음과 같이 사용될 수 있다.
- Role이 설정된 네임스페이스에 RoleBinding(Role과 RoleBinding은 같은 네임스페이스에 존재해야한다.)
- ClusterRole을 특정 네임스페이스에 RoleBinding

만약 ClusterRole을 모든 네임스페이스에 적용하고 싶다면 ClusterRoleBinding을 사용해야한다.

### RoleBinding Example

```
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "jane" to read pods in the "default" namespace.
# You need to already have a Role named "pod-reader" in that namespace.
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# You can specify more than one "subject"
- kind: User
  name: jane # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

`pod-reader`라는 Role을 `jane` User에게 부여한다.

```
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "dave" to read secrets in the "development" namespace.
# You need to already have a ClusterRole named "secret-reader".
kind: RoleBinding
metadata:
  name: read-secrets
  #
  # The namespace of the RoleBinding determines where the permissions are granted.
  # This only grants permissions within the "development" namespace.
  namespace: development
subjects:
- kind: User
  name: dave # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

ClusterRole의 경우도 RoleBinding할 수 있다.


### ClusterRoleBinding example

```
apiVersion: rbac.authorization.k8s.io/v1
# This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

ClusterRoleBinding은 모든 네임스페이스에 적용된다.

참고로 RoleBinding/ClusterRoleBinding의 경우 수정할 수 없으며 수정을 원하면 지운다음 다시 생성해야한다.
이렇게 설계된 이유는 다음과 같다.
- roleRef를 변경불가능하게 만들면 현재 바인딩되어 있는 Object의 권한을 업데이트할 수 있다. 그렇게하면 Subjects를 별도의 Subject의 할당된 Role의 변경없이 관리할 수 있다. (해당 Role을 업데이트하는 것 다른 Role을 대신 사용하지 않는다.)
- 다른 Role을 바인딩한다면 다른 RoleBinding이다. roleRef가 변경될 때마다 RoleBinding을 지우고 다시생성해야하는 것은 결국 새로운 Role을 부여하려는 의도이다.


## [Use Kubeadm to install a basic cluster](#ref2)

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

