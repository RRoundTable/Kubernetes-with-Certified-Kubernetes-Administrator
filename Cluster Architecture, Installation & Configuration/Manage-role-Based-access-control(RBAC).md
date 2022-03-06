# Manage role Based access control(RBAC)

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

## References

<a name="#ref1" href="https://kubernetes.io/docs/reference/access-authn-authz/rbac/"> [1] Using RBAC Authorization [websites] (2022, Feb, 13)</a>