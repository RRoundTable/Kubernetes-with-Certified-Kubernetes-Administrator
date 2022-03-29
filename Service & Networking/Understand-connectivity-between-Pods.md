
# Understand connectivity between Pods

본 장의 내용은 [블로그](#ref1)을 번역한 것이다.

## Pod 내부 Container의 Network

Pod는 하나이상의 Container로 구성되어 있으며 Network Stack이나 Volume을 공유한다.
예를 들어 하나의 Container가 nginx를 실행중이고 80 Port가 Listening중이면 다른 해당 Pod의 Container가 `localhost:80`으로 접근할 수 있는 것이다.

다음 이미지를 살펴보자.

![](https://miro.medium.com/max/1400/1*0Xo-WpbTTGKZhJt7TvFLZQ.png)

위에서 부터 살펴보면 `eth0`이라는 Physical Network Inferface가 있다.
여기에 `docker0`이라는 Bridge가 연결되어 있다.
`docker0`과 `veth0`은 같은 네트워크에 있다. (`172.17.0.0/24`)
`docker0`은 `172.17.0.1`을 할당받았으며 `veth0`의 Default Gateway이다.

Container가 실행될 때 Network Namespace가 설정되는 방식때문에 Container는 `veth0`만 바라본다.
외부와 통신할려고 할때는 `docker0`을 통해서 통신한다.



이제 다른 container를 실행시켜보자.

![](https://miro.medium.com/max/1400/1*ZdgIoY6tuOqK-r6wgL7d5A.png)

Container2도 `docker0`을 사용하며 Default Gateway로 `veth1`을 사용하는 것을 볼 수 있다.
`veth1`은 `172.17.0.3`을 할당받았다.
따라서 `docker0`, `veth0`, `veth1`이 모두 같은 논리적 네트워크에 존재한다.
그리고 모든 Container은 Bridge `docker0`을 통해서 통신한다. (서로 IP를 Discover할 수 있는한)

하지만 위의 그림은 Container가 Network Stack을 공유하는 것은 아니다.
Pod는 Network Stack을 공유하며 이는 아래 그림과 같다.

![](https://miro.medium.com/max/1400/1*akBBZKad2SAxSnJNaSHVmg.png)

위의 그림을 보면 `veth0`을 Container1, 2이 서로 공유하는 것을 볼 수 있다. (`172.17.0.2`)
따라서 Container1이 localhost로 접근해서 Container2와 통신할 수 있는 것이다.
또한 Container1과 Container2는 서로 같은 Port를 사용할 수 없다.

이렇게 설계되면 Container끼리 격리되는 동시에 Volume 혹은 Network를 통해서 서로 정보를 주고 받을 수 있다.

Kubernetes는 위의 패턴을 특별한 Container를 통해서 구현했다.
해당 Container의 유일한 기능은 다른 Container에게 Network Interface를 제공하는 것이다.
만약 동작중인 Kubernetes Cluster Node에서 `docker ps`를 입력해보면 하나의 `pause`명령어로 실행되고 있는 Container를 확인할 수 있을 것이다.

```
$ docker ps | grep pause

c691b686d9b5   602401143452.dkr.ecr.ap-northeast-2.amazonaws.com/eks/pause:3.1-eksbuild.1   "/pause"                 27 seconds ago   Up 26 seconds             k8s_POD_node-shell-31073f60-6772-4ebc-aeb1-51c53f9f3faf_kube-system_dfe2190d-da00-43a9-86d9-4bee4d40d720_0
```

`pause` 명령어는 현재 Process를 Signal을 받을때까지 Suspend하고 그 동안 Container들은 Sleep 상태를 유지한다.
Kubernetes가 SIGTERM Signal을 보내면 그때부터 동작한다.

`pause` Container가 하는 역할이 작아보이지만 실제로는 매우 중요한 역할을 한다.
Virtual Network Interface를 제공하며 Container들은 해당 Network Interface를 활용하여 다른 Container 그리고 외부와 통신한다.

![](https://miro.medium.com/max/1400/1*7JLi1Rl0G0FAeu-hiTGSGQ.png)


## Pod의 네트워크

Kubernetes Cluster는 하나 이상의 노드로 구성되어 있다.
노드는 물리적 혹은 가상적 Host System이다.
노드는 Container Runtime을 가지고 있어야하며 이에 대한 의존성도 설치되어있다. (Docker)
그리고 몇몇 Kubernetes System Component는 서로 Network로 연결되어 있으며 Cluster내 다른 노드에 접근을 가능하게 한다.
간단한 두개의 노드로 구성된 Cluster는 아래 모습과 같다.

![](https://miro.medium.com/max/1400/1*XGG8e2tbP4bQbsS33gfwUw.png)

위의 그림에서는 Private Network를 가정했다. (`10.100.0.0/24`)
Router는 `10.100.0.1`을 주소를 가지고 `eth0`, `eth1`는 각각 `10.100.0.2`, `10.100.0.3`주소를 가진다.

아래 그림은 위의 그림에서 Pod를 구체화하여 표현한 것이다.
Pod는 위의 Private Network를 사용하지 않는다는 것을 기억하자.

![](https://miro.medium.com/max/1400/1*RiLtoAdCfcJygwePVJzZOA.png)

Host는 `eth0` Interface를 `10.100.0.2`로 유지하며 Default Gateway는 `10.100.0.1`의 Router이다.
`veth0`은 Pause Container와 함께 생성되며 Container1, 2와 Network Stack을 공유한다.

Bridge가 생성될 때 Local Rounting Rule이 생성되기 때문에, `eth0` `172.17.0.2`에 도착한 패킷은 Bridge(`docker0`)로 포워딩된다.
그리고 `veth0`으로 최종 포워딩된다.

만약 우리가 Pod가 해당 호스트에서 `172.17.0.2`에서 동작하는 것을 안다면, Router에 `10.100.0.2`가 `172.17.0.2`로 HoP되는 것을 추가할 수 있다.

이제 다른 호스트를 살펴보자.
오른쪽 호스트는 `eth0`의 주소가 `10.100.0.3`이고 Default Gateway는 `10.100.0.1`로 왼쪽 호스트와 동일하다.
마찬가지로 `docker0` Bridge로 연결되어 있으며 Bridge주소는 `172.17.0.2`이다.
이는 왼쪽 호스트와 동일하며 위의 그림 예시는 의도적으로 안좋은 케이스를 시각화한것이다.

하나의 노드는 다른 노드의 Bridge가 어떤 Private Address Space를 할당받았는지 알아야한다.
이를 위해서 Kubernetes는 두 가지 방법으로 해당구조를 제공한다.

![](https://miro.medium.com/max/1400/1*oyGbXt7kStLd85ZT4it3oQ.png)

1. 각 노드에서 Bridge에 Address Space를 할당
2. Gateway에 Rounting Rule 추가

Virtual Network Interface와 Bridge, Routing Rule의 조합을 Overlay Network라 한다.
위의 이미지르르 보면 `docker0`대신 `cbr0` (Custom Bridge)으로 대체되었으며 Private Address Space를 받았다.
마찬가지로 `veth0`도 Private Address Space를 할당받은 것을 알 수 있다.

# Reference

<a name="#ref1" href="https://medium.com/google-cloud/understanding-kubernetes-networking-pods-7117dd28727"> Understanding kubernetes networking: pods </a>


<a name="#ref2" href="http://ifeanyi.co/posts/linux-namespaces-part-4/"> A deep dive into Linux namespaces, part 4 </a>

<a name="#ref3" href="https://www.ianlewis.org/en/almighty-pause-container"> The Almighty Pause Container </a>
