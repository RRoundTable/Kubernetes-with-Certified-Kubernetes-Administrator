
# Know how to configure and use CoreDNS


## About CoreDNS

CoreDNS는 Kuberntes에서 Cluster DNS역할을 하는 Flexible, Entensible DNS Server이다.
CoreDNS는 CNCF에서 개발되었다.

Kube-dns대신에 CoreDNS를 사용할 수 있다.


## Installing CoreDNS

[해당 링크](https://github.com/coredns/deployment/tree/master/kubernetes)를 참고한다.

## Migrating to CoreDNS

Kubernetes 1.21 version에서는 kubeadm은 kube-dns를 사용하지 않는다.
kubeadm v1.23에서는 CoreDNS만 제공한다.

kubeadm에서 CoreDNS로 이동하기 위해서는 kubeadm을 사용하여 Cluster를 Upgrade한다.
이 경우 kubeadm이 CoreDNS Configuration을 생성한다. (기존 kube-dns ConfigMap을 보존한다.)


## Upgrading CoreDNS

[버전정보](#ref2)를 참고한다.


## DNS 서비스 사용자 정의하기

CoreDNS를 Deployment형태로 배포하였다면  Service를 활용하여 Static IP로 노출되었을 것이다.
Kubelet은 DNS Resolver정보를 `--cluster-dns=<dns-service-ip>`플래그로 넣을 수 있다.

DNS 이름 또한 Domain이 필요하다.
Kubelet에 대한 설정은 `--cluster-domain=<default-local-domain>`플래그로 설정한다.

DNS Server는 다음과 같은 것을 제공한다.
- Forward Lookups를 제공한다. (A, AAAA records)
- Port Lookup (SRV records)
- Reverse IP Address Lookup (PTR records)

만약 Pod의 `dnsPolicy`가 `default`로 설정되어 있다면 Pod는 Name Resoultion 설정을 노드(Pod가 동작중인)로 상속받는다.

이렇게 하고 싶지 않다면 kubelet을 `--resolv-conf` 플래그로 사용해야한다.

### CoreDNS

#### CoreDNS ConfigMap Options

CoreDNS는 모듈화 가능하고 플러그화 가능한 DNS Server이다.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s # 5초 이상 Health 신호가 없으면 Unhealthy
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }    
```

위의 ConfigMap에는 다음과 같은 플러그인이 있다.

- `errors`: 에러를 stdout으로 로깅
- `health`: `http://localhost:8080/health`로 CoreDNS의 Health 정보 보고
- `ready`: 모든 플러그인이 Readiness신호를 보냈으면 Port 8081 HTTP Endpoint로 200 Ok 반환
- `kubernetes`: CoreDNS는 DNS Query에 대해서 쿠버네티스 Pod와 Service에 대한 IP정보를 기반으로 대응함.
- `prometheus`: CoreDNS에 Metric을 수집
- `forward`: 쿠버네티스 Cluster Domain에 포함되지 않는 Query를 미리 정의된 Resolvers로 보냄.
- `cache`: Frontend Cache를 가능하게 함
- `loop`: 간단한 Forwarding Loop를 탐지.Loop 발견시 CoreDNS halt
- `reload`:  Corefile 변경시 자동으로 Reload
- `loadbalance`: RoundRobin DNS Loadbalancer


## 서비스 및 파드용 DNS

Kubernetes는 Service와 Pod의 DNS를 관리하며 개별 Container들이 DNS를 해석할 때 DNS 서비스를 사용하도록 kubelet을 설정한다.

### 서비스 네임스페이스

Namespace를 지정하지 않을 경우 Pod의 Namespace에 한정되지만 Namespace를 지정하면 다른 Namespace에 도달할 수 있다.

Pod의 `/etc/resolv.conf`를 살펴보면 쉽게 알 수 있다. nameserver는 `coredns` 혹은 `kube-dns`의 Pod IP이다.
search는 Default Domain의 역할을 하며 Domain을 지정하지 않으면 자동으로 합쳐진다.
즉, Namespace를 지정하지 않으며 search에 해당하는 Domain이 붙고 이는 Pod Namespace에 한정되는 것을 의미한다.

```
# /etc/resolv.conf
nameserver 10.32.0.10
search <namespace>.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- A/AAAA 레코드: Normal Service
`my-svc.my-namespace.svc.cluster-domain.example`

`mlflow.mlflow.svc.local`

- SRV 레코드: [Headless Service](https://kubernetes.io/ko/docs/concepts/services-networking/service/#%ED%97%A4%EB%93%9C%EB%A6%AC%EC%8A%A4-headless-%EC%84%9C%EB%B9%84%EC%8A%A4)
`_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster-domain.example`




### 파드 네임스페이스

- A/AAAA 레코드

`pod-ip-address.my-namespace.pod.cluster-domain.example`

`172-17-0-3.default.pod.cluster.local.`

Deployemnt 혹은 DeamonSet에 의해 생성된 Pod는

`pod-ip-address.deployment-name.my-namespace.svc.cluster-domain.example`


#### Pod의 hostname 및 subdomain

Pod가 생성되면 Hostname은 해당 Pod의 `metadata.name`이 된다.

또한 Pod Spec에서 `hostmname`을 명시적으로 지정할 수 있다.
참고로 `hostname`이 `metadata.name`보다 우선한다.

아래 예시는 Hostname은 `busybox-1`이다.

```
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostname: busybox-1
  subdomain: default-subdomain
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox
```

또한 Pod Spec에서 `subdomain`을 지정할 수 있다.
아래 예시에서 Hostname은 `busybox-2`이고 Subdomain은 `default-subdomain`이다.

Pod의 전체 주소 도메인은 다음과 같다. `busybox-2.default-subdomain.my-namespace.cluster-local.example`

```
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
  labels:
    name: busybox
spec:
  hostname: busybox-2
  subdomain: default-subdomain
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox

```

#### Pod setHostnameASFQDN 필드

Pod의 FQDN을 Pod의 Hostname이 되도록 설정할 수 있다.

`setHostnameASFQDN: true`로 설정하면 된다.

```
# Pod에 접속해서 아래 명령어 비교해보기
$ hostname
$ hostname --fqdn
```

#### Pod의 DNS

Pod Spec의 `dnsPolicy` 항목 참고.


- `Default`: 파드가 실행되고 있는 노드로부터 Resolution Configuration을 상속받는다.
- `ClusterFrist`: Cluster Domain Suffix 구성과 일치하지 않는 DNS 쿼리는 노드에서 상속된 업스트림 네임서버로 전달한다.
- `ClusterFristWithHostNet`: HostNetwork에서 실행중인 Pod는 해당 DNS 정책을 사용해야한다.
- `None`: 쿠버네티스 환경의 DNS 설정 무시

#### Pod의 DNS 설정

- `nameservers`
- `searches`
- `options`


## Reference

<a name="#ref1" href="https://github.com/coredns/deployment/tree/master/kubernetes">[1] CoreDNS [websites] (2022, Mar, 19)</a>

<a name="#ref2" href="https://github.com/coredns/deployment/blob/master/kubernetes/CoreDNS-k8s_version.md">[2] CoreDNS version in Kubernetes [websites] (2022, Mar, 19)</a>

<a name="#ref3" href="https://kubernetes.io/ko/docs/tasks/administer-cluster/dns-custom-nameservers/">[3] DNS 서비스 사용자 정의하기 [websites] (2022, Mar, 19)</a>

<a name="#ref4" href="https://kubernetes.io/ko/docs/concepts/services-networking/dns-pod-service/">[4] 서비스 및 파드용 DNS [websites] (2022, Mar, 19)</a>