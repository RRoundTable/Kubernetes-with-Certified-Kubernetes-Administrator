
# Perform a version upgrade on a Kubernetes cluster using **Kubeadm**


이번 장에서는 Kubernetes Cluster를 업그레드하는 방법에 대해서 다룰 것이다.

## Before you begin

- [release note](#ref1)를 신중하게 읽어라
- Cluster는 Static Control Plane과 etcd pods 혹은 External etcd가 필요하다.
- 중요한 Component는 백업해라. `kbueadm upgrade`는 내부 Workloads에 영향주지 않지만 백업은 항상 중요하다.
- [Swap](#ref2)은 Disabled해야한다.

### Additional information

- kubelet MINOR verssion 업그레이드 이전에 Node를 Draining하는 것이 필요하다. Control Plane의 경우 CoreDNS Pods 혹은 다른 중요한 Workloads를 실행시켜야한다.
- 모든 Container는 upgrade 이후 restart한다. Container Spec Hash 값이 변경되었기 때문이다.


## Determine which version to upgrade to

```
apt update
apt-cache madison kubeadm
# find the latest 1.20 version in the list
# it should look like 1.20.x-00, where x is the latest patch
```

## Upgrading control plane nodes

Control Plane에 대한 업그레이드는 하나의 노드에서 실행되어야합니다.
하나의 첫번째로 업그레이드하고 싶은 Control Plane을 선택하고 먼저 업그레이드합니다.
해당 Control Plane은 `/etc/kubernetes/admin.conf`을 가지고 있어야합니다.


### 첫 번째 Control Plane

- kubeadm upgrade
  
    ```
    # replace x in 1.20.x-00 with the latest patch version
    apt-mark unhold kubeadm && \
    apt-get update && apt-get install -y kubeadm=1.20.x-00 && \
    apt-mark hold kubeadm
    -
    # since apt-get version 1.1 you can also use the following method
    apt-get update && \
    apt-get install -y --allow-change-held-packages kubeadm=1.20.x-00
    ```
- kubeadm 버전확인

    ```
    kubeadm version
    ```

- upgrade plan 검증

    ```
    <!-- This command checks that your cluster can be upgraded, and fetches the versions you can upgrade to. It also shows a table with the component config version states. -->
    kubeadm upgrade plan
    ```

    **참고**

    - `kubeadm upgrade`는 Default 설정상 기존 Certificate를 대체하고 업데이트합니다. `--certificate-renewal=false` 해당 플레그를 사용하면 기존 Certificate를 사용할 수 있습니다.
    - `kubeadm upgrade plan`가 Manual Upgrade에 필요한 Component Configs를 보여준다면 사용자는 `kubeadm upgrade apply`를 적용하기 위해서 config file을 `--config` 플래그로 제공해야한다.

- 업그레이드 하고 싶은 버전을 선택한 후 적절한 명령어를 입력한다.
  ```
    # replace x with the patch version you picked for this upgrade
    sudo kubeadm upgrade apply v1.20.x
  ```

  명령어 완료시 아래와 같은 메세지가 보인다.

  ```
    [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.20.x". Enjoy!

    [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
  ```

- CNI Provider plugin을 업그레이드 한다.


### 다른 Control Plane

```
sudo kubeadm upgrade node
```


###  Drain the node

노드가 업그레이드하는 동안 해당 노드에 Workload가 스케쥴링 되지 않도록 설정한다.

```
# replace <node-to-drain> with the name of your node you are draining
kubectl drain <node-to-drain> --ignore-daemonsets
```


### Upgrade kubelet and kubectl

```
 # replace x in 1.20.x-00 with the latest patch version
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.20.x-00 kubectl=1.20.x-00 && \
apt-mark hold kubelet kubectl
-
# since apt-get version 1.1 you can also use the following method
apt-get update && \
apt-get install -y --allow-change-held-packages kubelet=1.20.x-00 kubectl=1.20.x-00
    
```

```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Uncordon the node

다시 스케쥴링 가능하게 변경합니다.

```
# replace <node-to-drain> with the name of your node
kubectl uncordon <node-to-drain>
```

## Upgrade worker nodes

Worker Node에 대한 업그레이드는 동시에 진행되어야하며 기존 Workload를 실행하는데 필요한 최소환의 Capacity를 보장해야합니다.

### Upgrade kubeadm

```
# replace x in 1.20.x-00 with the latest patch version
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.20.x-00 && \
apt-mark hold kubeadm
-
# since apt-get version 1.1 you can also use the following method
apt-get update && \
apt-get install -y --allow-change-held-packages kubeadm=1.20.x-00
```

### Call kubeadm upgrade

```
sudo kubeadm upgrade node
```

### Drain the node

```
# replace <node-to-drain> with the name of your node you are draining
kubectl drain <node-to-drain> --ignore-daemonsets
```

### Upgrade kubelet and kubectl 

```
# replace x in 1.20.x-00 with the latest patch version
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.20.x-00 kubectl=1.20.x-00 && \
apt-mark hold kubelet kubectl
-
# since apt-get version 1.1 you can also use the following method
apt-get update && \
apt-get install -y --allow-change-held-packages kubelet=1.20.x-00 kubectl=1.20.x-00
```

kubelet 재시작

```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Uncordon the node

```
# replace <node-to-drain> with the name of your node
kubectl uncordon <node-to-drain>
```


## Verify the status of the cluster 

업그레이드 이후 Cluster 상태확인

```
kubectl get nodes
```

노드의 `STATUS`가 `Ready`상태인지 확인한다.

## Recovering from a failure state

`kubeadm upgrade`가 실패하고 Rollback이 안될때 예를 들어 예상치못한 Shutdown이 있을 때 다시 `kubeadm upgrade`를 실행하면 정상작동할 것이다.

비정상상태에서 회복하기 위해서는 `kubeadm upgrade apply --force`를 해볼수도 있다.

kubeadm upgrade동안 `/etc/kubernetes/tmp`에 백업이 생긴다.
- `kubeadm-backup-etcd-<date>-<time>`
- `kubeadm-backup-manifests-<date>-<time>`

`kubeadm-backup-etcd-<date>-<time>`
해당 Control Plane에 로컬 etcd member data를 가지고 있다.
etcd upgrade가 실패 혹은 롤백이 작동하지 않을 때 이 폴더의 파일들은 `/var/lib/etcd`에 복원된다.
Exteranl etcd의 경우 이 백업폴더는 비워질 것이다.

`kubeadm-backup-manifests-<date>-<time>`
Static Pod Manifest에 대해서 백업이 생긴다.
etcd upgrade가 실패 혹은 롤백이 작동하지 않을 때 `/etc/kubernetes/manifests`에 자동으로 복구될 것이다.
만약 업그레이드 전과 후가 차이가 없다면 해당 백업은 생성되지 않는다.

## How it works

- `kubeadm upgrade apply`는 다음과 같이 진행한다.
  - Cluster가 업그레이드 가능상태인지 확인
    - API server가 Reachable한지
    - 모든 노드가 `Ready`상태인지
    - Control Plane이 Healthy인지
  - Version Skew Policy 강제
  - Control Plane 이미지가 접근이 가능한지 확인
  - Component Config가 버전 업그레이드가 필요하다면 Replacements를 생성하거나 사용자가 제공한 Config를 사용한다.
  - Control Plane Componet 업그레이드 혹은 롤백
  - 새로운 `kube-dns`, `kube-proxy` Apply 그리고 필요한 RBAC가 모두 생성되었는지 확인
  - 새로운 Certificate와 API server를 위한 Key files를 생성 그리고 이전 파일을 백업(180일 이후 만료된다면)

- `kubeadm upgrade node`는 첫번째 Control Plane이 아닌 다른 Control Plane Node에 다음과 같이 적용된다.
  - Cluster로 부터 `ClusterConfiguration`을 패치한다.
  - kube-apiserver certificate를 선택적으로 백업한다.
  - Control Plane을 위한 Static Pod Manifest를 업그레이드한다.
  - 해당 노드에 적합하게 kubelet을 업그레이드한다.
  
- `kubeadm upgrade node`는 Worker Node에 다음과 같이 적용된다.
  - Cluster로 부터 `ClusterConfiguration`을 패치한다.
  - 해당 노드에 적합하게 kubelet을 업그레이드한다.


## References

<a name="#ref1" href="https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.23.md"> [1] kubernetes release notes [websites] (2022, 03, 06) </a>

<a name="#ref2" href="https://www.baeldung.com/linux/swap-space-use"> [1] The Use of Swap Space in Modern Linux Systems [websites] (2022, 03, 06) </a>
