
# Manage a highly available Kubernetes cluster

본 포스트는 `kubeadm`을 통해서 Cluster를 구축할 수 있는 두 가지 방법에 대해서 서술할 것이다.
- Stacked Control Plane Node를 통한 방법: etcd와 Control Plane은 같이 있다.
- External etcd Cluster: etcd와 Control Plane은 분리되어 있다.

[Stacked ETCD Topology](#ref2)
- Control Plane Node 다운시 etcd도 함께 다운되며, 다른 Control Plane과 정보가 일치하지 않게된다.

![](https://d33wubrfki0l68.cloudfront.net/d1411cded83856552f37911eb4522d9887ca4e83/b94b2/images/kubeadm/kubeadm-ha-topology-stacked-etcd.svg)

[External ETCD Topology](#ref2)
- Control Plane Node 다운되더라도 etcd는 영향이 없다. 하지만 더 많은 호스트가 필요하다.
![](https://d33wubrfki0l68.cloudfront.net/ad49fffce42d5a35ae0d0cc1186b97209d86b99c/5a6ae/images/kubeadm/kubeadm-ha-topology-external-etcd.svg)

## Container Images

Cluster를 구성하는 호스트는 Kubernetes Container Image Registry로 부터 모두 이미지를 읽고 패치할 수 있어야한다.
만약 호스트가 이미지 Pull 권한이 없는 상태에서 Highly-Available Clsuter를 배포하고 싶다면 다른 수단을 고려해야한다.

## Command Line Interface

`kubectl` 필요

## First steps for both methods

### Create load balancer for kube-apiserver

1. `kube-apiserver` Loadbalancer 생성하기 (DNS에 Name이 등록되어야한다.)
   1. 클라우드 환경의 경우 Control Plane을 TCP Forwarding Loadbalancer 뒤에 두어야한다. 이 Loadbalancer는 모든 Health Control Plane Node에게 트래픽을 분배한다. `kube-apiserver`는 Control Plane의 health check정보를 Listen한다.(default: 6443) 
   2. 참고로 IP Address를 직접적으로 사용하는 것은 클라우드 환경에서 권장되지 않는다. 
   3. Loadbalancer는 `kube-apiserver`의 Port를 통해서 모든 Control Plane Node와 통신가능해야한다. 
   4. 항상 Loadbalancer의 주소와 kubeadm의 `ControlPlaneEndpoint`와 일치하도록 한다.
   5. [Options for Software Load Balancing](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#options-for-software-load-balancing)
2. 첫번째 Control Plane Node에 Loadbalancer를 추가한다. 그리고 Connection을 확인한다.
   ```
   <!-- A connection refused error is expected because the API server is not yet running. A timeout, however, means the load balancer cannot communicate with the control plane node. If a timeout occurs, reconfigure the load balancer to communicate with the control plane node. -->

   nc -v <LOAD_BALANCER_IP> <PORT>
   ```

3. 나머지 Control Plane Node를 추가한다.

## Stacked control plane and etcd nodes

1. Control Plane 초기화
   ```
    sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs
   ```
   - `--kubernetes-version`: kubeadm, kubelet, kubectl, Kerernetes가 모두 일치하는 것을 권장
   - `--control-plane-endpoint`: Address or DNS and Port of Loadbalance
   - `--upload-certs`: Control Plane간 서로 공유되어야하는 Certificate 정보.

    ```
    # 아래와 같은 유형의 출력이 된다. 이 출력을 저장하고 worker node가 cluster에 join시 활용해야한다.
    ...
    You can now join any number of control-plane node by running the following command on each as a root:
        kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07

    Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
    As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.

    Then you can join any number of worker nodes by running the following on each as root:
        kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
    ```

2. CNI Plugin을 적용
   [Installing a Pod network add-on](#ref4)

3. Test
   ```
   kubectl get pod -n kube-system -w
   ```
4. 처음 Control Plane을 제외한 나머지 Node들 Cluster에 추가하기
   ```
   sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07
   ```
   - `--control-plane`: 해당 플레그는 Control Plane을 추가하는 것을 의미한다.
   - `--certificate-key ...`: 클러스터내에 Secret형태로 Control Plane Certificate가 다운로드된다.


## External etcd nodes

### Set up the etcd cluster

1. [etcd cluster를 설정한다.](#ref5)
2. [SSH 설정을 한다.](#ref6)
3. etcd 노드로 부터 아래 파일을 복사하여 첫 번째 Control Plane에 넣는다.
    ```
    export CONTROL_PLANE="user@host"
    scp /etc/kubernetes/pki/etcd/ca.crt "${CONTROL_PLANE}":
    scp /etc/kubernetes/pki/apiserver-etcd-client.crt "${CONTROL_PLANE}":
    scp /etc/kubernetes/pki/apiserver-etcd-client.key "${CONTROL_PLANE}":
    ```

### Set up the first control plane node

1. `kubeadm-config.yaml`을 아래와 같이 만든다.

```
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" # change this (see below)
etcd:
  external:
    endpoints:
      - https://ETCD_0_IP:2379 # change ETCD_0_IP appropriately
      - https://ETCD_1_IP:2379 # change ETCD_1_IP appropriately
      - https://ETCD_2_IP:2379 # change ETCD_2_IP appropriately
    caFile: /etc/kubernetes/pki/etcd/ca.crt
    certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
    keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
```

아래 항목들은 변경해줘야한다.
- LOAD_BALANCER_DNS
- LOAD_BALANCER_PORT
- ETCD_0_IP
- ETCD_1_IP
- ETCD_2_IP

2. kubeadm init
   ```
   sudo kubeadm init --config kubeadm-config.yaml --upload-certs
   ```
   Stacked control plane and etcd nodes에서 Control Plane하는 것과 유사한 과정을 거침

3. 처음 Control Plane을 제외한 나머지 Node들 Cluster에 추가하기(Stacked etcd setup 참고)
   - `--certificate-key`는 default로 두 시간후면 만료되니 해당 시간내에 작업하는 것을 권장한다.



## External etcd nodes

<a name="#ref1" href="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/"> [1] Creating Highly Available Clusters with kubeadm [websites] (2022, Mar, 6)</a>

<a name="#ref2" href="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#stacked-etcd-topology"> [2] Stacked etcd topology [websites] (2022, Mar, 6)</a>

<a name="#ref3" href="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#external-etcd-topology"> [3] External etcd topology [websites] (2022, Mar, 6)</a>

<a name="#ref4" href="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network"> [4] Installing a Pod network add-on [websites] (2022, Mar, 6)</a>

<a name="#ref5" href="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/"> [5] Set up a High Availability etcd Cluster with kubeadm [websites] (2022, Mar, 6)</a>

<a name="#ref6" href="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#manual-certs"> [6] Manual certificate distribution [websites] (2022, Mar, 6)</a>

