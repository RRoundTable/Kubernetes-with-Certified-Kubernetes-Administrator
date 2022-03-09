# Operating etcd clusters for Kubernetes

etcd는 일관성있으며 매우 높은 사용성을 가진 Kubernetes에서 사용하는 Key-Value Store이다.
만약 Cluster가 etcd를 Backing Store로 사용한다면 Cluster data에 대한 Backup Plan이 있는 것이다.

## Before you begin

- Kubernetes Clsuter
- kubectl


## Prerequisites

- etcd를 홀수의 구성을 가진 cluster로 실행하라
- etcd는 Leader-Based Distributed System이다. Leader가 주기적으로 Heartbeats를 Follower에게 보내도록 하라.
- Resource Starvation이 안일어나도록하라
- etcd Cluster가 안정적이도록 하는 것은 Kubernetes Cluster안정성에 중요한 요소이다. 그러므로 etcd Cluster를 독립적인 환경에서 운영하라.
- 프로덕션에서 etcd의 권장버전은 3.2.10+.이상이다.

## Resource requirements

[etcd hardware configuration](#ref4)

## Starting etcd clusters

### Single-node etcd cluster

1. 아래의 명령어 실행
   ```
   etcd --listen-client-urls=http://$PRIVATE_IP:2379 \
        --advertise-client-urls=http://$PRIVATE_IP:2379
   ```
2. Kubernetes API server를 `--etcd-servers=$PRIVATE_IP:2379` 플래그를 사용하여 실행


### Multi-node etcd cluster

프로덕션 환경에서 지속가능성과 높은 사용성을 위해서 etcd를 Multi-Node cluster로 운영하는 것을 권장한다.
보통 5개의 노드로 이루어진 Cluster를 권장한다.

etcd Cluster를 구축할 때 [해당자료](#ref5)를 참고하라.


1. 아래의 명령어 실행
   ```
   etcd --listen-client-urls=http://$IP1:2379,http://$IP2:2379,http://$IP3:2379,http://$IP4:2379,http://$IP5:2379 --advertise-client-urls=http://$IP1:2379,http://$IP2:2379,http://$IP3:2379,http://$IP4:2379,http://$IP5:2379
   ```
2. Kubernetes API server를 `--etcd-servers=$IP1:2379,$IP2:2379,$IP3:2379,$IP4:2379,$IP5:2379` 플래그를 사용하여 실행


### Multi-node etcd cluster with load balancer

etcd Cluster에서 로드벨런서를 운영하기 위해서 아래의 과정을 거쳐야한다.

1. etcd Clsuter 구축
2. etcd Cluster에 로드벨런서 설정. 예를 들어 로드벨런서의 주소를 `$LB`라고 한다.
3. Kubernetes API Server를 `--etcd-servers=$LB:2379` 해당 플래그로 실행한다.

## Securing etcd clusters

etcd에 접근하는 것은 Kubernetes Cluster에 Root Permission과 동일한 권한이다.
따라서 이상적으로는 API Server만 etcd에 접근할 수 있어야한다.
etcd 데이터의 민감성을 고려해보면 etcd Cluster에 접근해야하는 노드에만 Permission을 부여하는 것이 권장된다.

etcd의 보안을 강화하기 위해서 Firewall 혹은 etcd에서 제공하는 Security Feature를 사용할 수 있다.
etcd Security Feature는 x509 Public Key에 의존적이다.
시작하기 위해서 Key와 Certificate Pair를 만듬으로서 Secure Communication Channels를 구축한다.
예를 들어 `peer.key`, `peer.cert`의 key-pair를 etcd Member간에 communication을 위해서 사용한다.
`client.key`, `client.cert`는 etcd와 etcd의 클라이언트 통신에 사용한다.


### Securing communication

etcd peer 통신을 위해서는 `--peer-key-file=peer.key`와 `--peer-cert-file=peer.cert`이 필요하다.

client 통신을 위해서는 `--key-file=k8sclient.key`와 `--cert-file=k8sclient.cert`이 필요하다.
아래는 client 통신을 위한 예시이다.
```
ETCDCTL_API=3 etcdctl --endpoints 10.2.0.9:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  member list

```


### Limiting access of etcd clusters

보안을 강화하기 위해서 etcd Cluster에 접근을 제한해야한다. (Kubernetes API server만 허용)
TLS authentication을 사용한다.

예를 들어 `k8sclient.key`, `k8sclient.cert`의 key-pair가 있으며 이는 CA `etca.ca`에 의해서 보증된다.
etcd가 TLS를 통해서 `--client-cert-auth`로 설정된다면 이는 클라이언트로부터 제공된 Certificate를 System CA나 `--trusted-ca-file=etcd.ca`로 부터 전달된 CA로 검증할 것이다.
`--client-cert-auth=true`와 `--trusted-ca-file=etcd.ca`는 클라이언트의 접근을 `k8sclient.cert`로 제한할 것이다.

etcd가 정상적으로 설정되었다면 검증된 Certificate를 가진 클라이언트만 접근할 수 있다.
Kubernetes API Server가 접근할 수 있게하려면 `--etcd-certfile=k8sclient.cert`, `--etcd-keyfile=k8sclient.key`, `--etcd-cafile=ca.cert`로 설정해야한다.



## Replacing a failed etcd member

etcd Cluster는 맴버의 마이너한 실패에도 높은 안정성을 확보할 수 있다.
그러나 etcd Cluster의 안정성을 위해서 비정상 맴버를 다른 것으로 교체하는 것이 필요하다.
만약 복수의 맴ㅂ가 실패하면 하나씩 교체해나가야 한다. (제거한 후 추가한다.)

etcd가 고유한 맴버 ID를 가지고 있지만 실수를 방지하는 차원에서 맴버의 이름또한 고유하게 설정할 것을 권장한다.
예를 들어 `member1=http://10.0.0.1`, `member2=http://10.0.0.2`, `member3=http://10.0.0.3`에서 member1이 실패할 경우 `member4=http://10.0.0.4`로 교체해준다.

1. 실패한 맴버의 member ID 확인
    ```
    etcdctl --endpoints=http://10.0.0.2,http://10.0.0.3 member list
    ```

    ```
    # expected output
    8211f1d0f64f3269, started, member1, http://10.0.0.1:2380, http://10.0.0.1:2379
    91bc3c398fb3c146, started, member2, http://10.0.0.2:2380, http://10.0.0.2:2379
    fd422379fda50e48, started, member3, http://10.0.0.3:2380, http://10.0.0.3:2379
    ```

2. 실패한 맴버 제거

    ```
    etcdctl member remove 8211f1d0f64f3269
    ```

    ```
    # expected output
    Removed member 8211f1d0f64f3269 from cluster
    ```

3. 새로운 맴버 추가

    ```
    etcdctl member add member4 --peer-urls=http://10.0.0.4:2380
    ```

    ```
    # expected output
    Member 2be1eb8f84b7f63e added to cluster ef37ad9dc622a7c4
    ```


4. IP가 `10.0.0.4`인 머신에서 새로운 맴버 시작

    ```
    export ETCD_NAME="member4"
    export ETCD_INITIAL_CLUSTER="member2=http://10.0.0.2:2380,member3=http://10.0.0.3:2380,member4=http://10.0.0.4:2380"
    export ETCD_INITIAL_CLUSTER_STATE=existing
    etcd [flags]
    ```

5. 아래의 것 실행
   1. `--etcd-servers`플래그로 Kubernetes API Server 실행, Kubernetes가 etcd 설정변화를 알게한다. 그리고 API Server를 재실행한다.
   2. 로드벨런서 설정 업데이트


## Backing up etcd Clsuter

모든 Kubernetes Object는 etcd에 저장된다.
주기적으로 etcd cluster 데이터를 백업하는 것은 예끼치 못한 재앙에서 Kuberntes Clsuter를 복원하는데 중요하다. (모든 Control Plane이 없어졌다고 생각해보자..)
Snapshot file은 Kubernetes state를 포함한 중요한 정보를 가지고 있다.
Kubernetes 정보를 안전하게 지키기 위해서 Snapshot file를 암호화한다.

### Built-in snapshot

etcd는 built-in snapshot을 제공한다.
실행중인 맴버의 Snapshot을 저장하기 위해서는 `etcdctl snapshot save` 명렬어를 사용한다.
또한 etcd data directory를 복사하여 `/member/snap/db` 만들수도 있다. (이는 현재 etcd 프로세스에서 사용중이지 않다.)
Snapshot은 기존 맴버의 성능에 영향을 주지 않는다.

```
ETCDCTL_API=3 etcdctl --endpoints $ENDPOINT snapshot save snapshotdb
```

다음 명령어로 snapshot을 검증할 수 있다.

```
ETCDCTL_API=3 etcdctl --write-out=table snapshot status snapshotdb
```

```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| fe01cf57 |       10 |          7 | 2.1 MB     |
+----------+----------+------------+------------+

```

### Volume snapshot

etcd가 Storage Volume을 사용중이라면 이를 복사하여 백업할 수 있다.


### Snapshot suing etcdctl optino

Snapshot 저장시 다양한 옵션이 존재한다.

```
# 옵션확인하기
ETCDCTL_API=3 etcdctl -h 
```

예를들어 Endpoint를 특정하여 Snapshot을 저장할 수 있다.

```
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file> \
  snapshot save <backup-file-location>
```
`trusted-ca-file`, `cert-file`, `key-file` etcd Pod로 부터 얻는 정보이다.


## Scaling up etcd clusters

etcd Cluster를 Scaling up하는 것은 성능을 저하시키지만 안정성을 강화한다.
Scaling은 Cluster 성능 혹은 Capability를 증가시키지 않는다.
일반적인 권장사항은 etcd Cluster를 Scaling Up 혹은 Down시키지 않는 것이다.
etcd Cluster에 Auto Scaling을 적용하지 마라.
일정수의 맴버를 유지하는 것을 권장한다.

가장 합리적인 수는 5이다.

## Restoring an etcd cluster

etcd는 Snapshot을 활용하여 복구하는 기능을 제공한다.

복구기능을 시작하기 전에 Snapshot이 존재해야한다.
이전 백업 활동 혹은 data directory에 남아있는 Snapshot file을 활용하여 복구한다.

```
# 이전 백업 활동 Snapshot
ETCDCTL_API=3 etcdctl --endpoints 10.2.0.9:2379 snapshot restore snapshotdb
```

```
# data directory
ETCDCTL_API=3 etcdctl --data-dir <data-dir-location> snapshot restore snapshotdb
```

만약 복구된 Cluster의 URL접근이 이전 Cluster에 의해서 변경되었다면 Kubernetes API Server는 이에 맞게 재설정되어야한다.
이 경우 Kubernetes API Server를 `--etcd-servers=$NEW_ETCD_Cluster`로 재시작해야한다.
만약 etcd Cluster에 로드벨런서를 사용했다면 로드벨런서도 적절히 업데이트해야하낟.

만약 etcd 맴버의 다수가 지속해서 실패한다면 etcd Cluster는 실패했다고 간주해야한다.
이런 시나리오에서 Kubernetes는 현재 상태에서 어떤 변화도 만들 수 없다.
스케쥴링된 Pod가 계속 실행하더라도 새로운 Pod는 스케쥴링될 수 없다.
이런 경우 etcd Cluster를 복구하고 Kubernetes API Server를 재설정해야한다.


**참고**

Kubernetes API Server가 실행중이라면 etcd cluster를 복구해서는 안된다.
대신 다음 절차를 따라라.

- 모든 API Server 중지
- etcd 복구
- API Server 재실행

그리고 `kube-scheduler`, `kube-controller-manager`, `kubelet`을 모두 재시작할 것을 권장한다.

## References

<a name="#ref1" href="https://zgundam.tistory.com/197"> ETCD Backup and Restore [website] (2022, Mar, 6) </a>

<a name="#ref2" href="https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster"> Backing up an etcd cluster [website] (2022, Mar, 6) </a>


<a name="#ref3" href="https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#restoring-an-etcd-cluster"> Restoring an etcd cluster [website] (2022, Mar, 6) </a>


<a name="#ref4" href="https://etcd.io/docs/v3.5/op-guide/hardware/#example-hardware-configurations"> Example hardware configurations [website] (2022, Mar, 6) </a>

<a name="#ref5" href="https://etcd.io/docs/v3.5/op-guide/clustering/"> Clustering Guide [website] (2022, Mar, 6) </a>


https://etcd.io/docs/v3.5/op-guide/clustering/