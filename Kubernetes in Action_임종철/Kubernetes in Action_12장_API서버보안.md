# Kubernetes in Action

---
---
## 12장 쿠버네티스 API 서버 보안

---
---
### 12.1 인증
* API 서버는 하나 이상의 인증 플러그인으로 구성될 수 있음
* 첫번째 플러그인은 사용자 이름, 사용자 ID, 클라이언트가 속한 그룹을 API 서버 코어에 반환함
	- API 서버는 나머지 인증 플러그인을 중지하고 '승인' 단계로 넘어감
* 여러 인증 플러그인
	- 클라이언트 인증서
	- HTTP 헤더로 전달된 인증 토큰
	- 기본 HTTP 인증
	- 기타

---
##### 12.1.1 사용자와 그룹
* 인증 플러그인은 인증된 사용자의 사용자 이름과 그룹을 반환
	- 해당 정보를 저장하지 않음
	- 사용자가 작업 수행 권한이 있는지 여부를 확인하는데 사용

##### 사용자
* 쿠버네티스는 API서버에 접속하는 두 가지 유형의 클라이언트를 구분
	- 실제 사람(즉, 사용자)
	- 포드(구체적으로, 포드 내부에서 실행하는 어플리케이션)
* 인증 플러그인을 통해 구별함
* 사용자는 SSO(Single Sign On) 시스템과 같은 외부 시스템을 통해 관리
	- 사용자 계정을 나타내는 어떤 리소스도 없음(API서버를 통해 사용자 생성/업데이트/삭제할 수 없음)
* 포드는 Service Accounts라는 매커니즘을 사용
	- 클러스터 내에 서비스어카운트 리소스로 생성되고 저장됨

##### 그룹
* 사용자 및 서비스어카운트는 모두 하나 이상의 그룹에 속할 수 있음
* 그룹은 그룹에 속한 사용자에게 한 번에 사용 권한을 부여하는데 사용
* 시스템에서 제공하는 기본 그룹
	- System: unauthenticated 그룹은 인증 플러그인이 어느 클라이언트도 인증할 수 없을 때 이 요청을 사용함
	- system: authenticated 그룹은 성공적으로 인증된 사용자에게 자동으로 할당함
	- system: serviceaccounts 그룹은 시스템의 모든 서비스어카운트를 포함함
	- system: serviceaccounts:<namespace>는 특정 네임스페이스의 모든 서비스어카운트를 포함함

---
#### 12.1.2 서비스어카운트 소개
* 서비스어카운트는 포드 내에서 실행되는 애플리케이션이 API 서버 자제와 인증하기 위한 방법
* 어플리케이션은 요청에 서비스어카운트의 토큰을 전달
	- 각 컨테이너는 /var/run/secrets/kubernetes.io/serviceaccount/token 에 '서비스어카운트의 인증 토큰'를 마운트함
	- 토큰을 사용해 API 서버에 연결하면
		+ 인증 플러그인이 서비스어키운트를 인증하고
		+ 서비스어카운트의 사용자 이름을 API 서버 코어로 다시 전달
* 서비스어카운트 사용자 이름 형식
`system:serviceaccount:<namespace>:<service account name>`
* API 서버는 구성된 사용자 승인 플러그인에게 이 '사용자 이름'을 전달
* 어플리케이션이 수행하려고 하는 동작이 서비스어카운트에 의해 허가되는지 여부를 결정
	
##### 서비스어카운트 리소스
* 서비스어카운트는 포드, 시크릿, ConfigMap 등과 같은 리소스이며 개별 네임스페이스로 범위가 지정됨
	- 각 네임스페이스의 디폴트 서비스어키운트는 자동으로 만들어짐(즉, 포드에 맞게 모두 생성됨)

Ex) 서비스어카운트 확인
```bash
$ kubectl get sa
NAME      SECRETS   AGE
default   1         2d2
```

![서비스어카운트](./Kubernetes in Action_12장_API서버보안/12장_00_서비스어카운트.png)
* 필요한 경우 서비스어카운트를 추가 가능
* 여러 포드에서 같은 서비스어카운트를 사용할 수 있음
	- 포드는 동일한 네임스페이스의 서비스어카운트만 사용 가능

##### 서비스어카운트가 인증과 어떻게 밀접하게 연관되어 있는지 이해하기
* 포드 매니페시트에서 계정 이름을 지정해 포드에 서비스어카운트를 할당 가능
	- 명시적으로 지정하지 않으면 네임스페이스의 디폴트 서비스어카운트를 사용
* 포드에 각기 다른 서비스어카운트를 할당하면 각 포드에 접근할 수 있는 리소스 제어 가능  
* 인증 토큰이 있는 요청이 API 서버에 수신 ->  서버는 토큰을 사용해 클라이언트를 인증 -> 관련 서비스어카운트가 요청된 작업을 수행할 수 있는지 여부를 결정
	- API서버는 클러스터 관리자가 구성한 시스템 전체 권한 플러스인에서 확인함
	- 사용 가능한 승인 플러그중 하나는 RBAC(역할 기반 접근 제어) 플러그인임

---
#### 12.1.3 서비스어카운트 생성
* 클러스터 보안 때문에 디폴트 서비스어카운트를 사용하지 않고 필요한 경우 서비스어카운트를 생성(추가)하여 사용
	- Ex) 클러스터 메타 데이터를 읽을 필요가 없는 포드는, 클러스터 리소스를 검색/수정할 수 없는 제한된 계정에서 실행해야 함 등

##### 서비스어카운트 생성
* `kubectL create serviceaccount` 사용
```bash
$ kubectl create serviceaccount foo
serviceaccount/foo created
```

Ex) kubectl describe로 서비스어카운트 확인하기
```bash
$ kubectl describe sa foo
Name:                foo
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>				# 이 서비스어카운트를 사용해 모든 포드에 자동으로 추가함
Mountable secrets:   foo-token-pqnfs	# 이 서비스어카운트를 사용하는 포드는 mountable secrets가 수행됬을 때, 이런 secret을 마운트 할 수 있음
Tokens:              foo-token-pqnfs	# 인증 토큰, 첫 번째 토큰이 컨테이너 내부에서 마운트됨	
Events:              <none>
```
* 사용자 지정 토큰 시크릿이 만들어지고 서비스어카운트와 연결됐음을 확인

Ex) 사용자 기반 수정된 서비스어카운트의 시크릿 확인
```bash
$ kubectl describe secret foo-token-pqnfs
Name:         foo-token-pqnfs
Namespace:    default
...
Data
====
ca.crt:     1025 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZ...
```
* 기본 서비스어카운트의 토큰과 동일한 항목(CA 인증서, 네임스페이스, 토큰)이 포함돼 있음을 알 수 있음
	- 기본 서비스어카운트와 토큰 자체는 다름
* 서비스어카운트에서 사용되는 인증 토큰은 JWT(JSON Web Token)

##### 서비스어카운트의 마운트할 수 있는 시크릿
* Mountable secrets
	- 포드의 서비스어카운트는 서비스 계정상의 Moutable Secrets에 나열된 시크릿만이 포드에 마운트되도록 허용함
	- 기능을 사용하려면 서비스어카운트는 다음과 같은 주석을 포함해야 함
		+ `kubernetes.io/enforce-mountable-secrets="true"`
		+ 주석을 사용하는 모든 포드는 서비스어카운트의 마운트 가능한 시크릿만 만들 수 있음(다른 시크릿은 사용 못함)

##### 서비스어카운트의 이미지가 시크릿을 가져오는 방식
* image pull Secrets
	- 개인용 이미지 스토리지에서 컨테이너 이미지를 가져오는 데 필요한 자격 증명을 가지고 있는 시크릿
Ex)image pull Secret이 포함된 서비스어카운트
```bash
$ cat sa-image-pull-secrets.yaml                                                20-06-12 - 11:33:10
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
imagePullSecrets:
- name: my-dockerhub-secret
```
* 서비스어카운트의 이미지 풀 Secrets는 마운트 가능한 시크릿과 약간 다르게 동작함
	- __서비스어카운트를 사용해 모든 포드에 자동으로 추가될 수 있는지를 결정__
	- 서비스어카운트에 이미지 풀 Secrets를 추가하면 각 포드에 개별적으로 추가할 필요가 없음

---
#### 12.1.4 포드에 서비스어카운트 할당
* 추가 서비스어카운트를 생성한 후에는 이를 포드에 지정해야 함
	- 포드 정의의 spec.serviceAccountName 필드에 서비스어카운트의 이름을 설정
```
참고 : 서비스어카운트는 포드가 생성될 때 반드시 지정돼야 한다. 그 이후에는 변경할 수 없다．
```

##### 사용자 지정 서비스어카운트를 사용하는 포드 생성
Ex) 사용자 지정 서비스어카운트를 사용한 포드(curl-custom-sa.yaml)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-custom-sa
spec:
  serviceAccountName: foo	# 이 포드는 default 대신에 foo 서비스어카운트를 사용함
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "9999999"]
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
```
* 앰베서더 컨테이너는 포드의 서비스어카운트의 토큰을 사용해 API 서버로 인증하기 위해 kubectl proxy 프로세스를 실행함

Ex) 포드의 컨테이너에 마운트된 토큰 확인하기
```bash
kubectl exec -it curl-custom-sa -c main -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/token
eyJhbGciOiJSUzI1NiIsImtpZCI6IjBFeVNkN2x...
```

##### API 서버와 통신하기 위해 서비스어카운트의 사용자 토큰 사용
* 앰배서더 컨테이너는 서버와 통신할 때 토큰을 사용
	- localhost:8001을 사용하는 앰베서더 컨테이너를 통해 토큰을 테스트 가능
	
Ex) 사용자 지정 서비스어카운트가 포함된 API 서버와 통신하기
```bash
$ kubectl exec -it curl-custom-sa -c main -- curl localhost:8001/api/v1/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/pods",
    "resourceVersion": "307842"
  },
  "items": [
    {
      "metadata": {
...
```
* 서버에서 적절한 응답을 받았으므로 사용자 지정 서비스어카운트가 포드를 나열할 수 있음
	- 클러스터 RBAC 인증 플러그인을 사용하지 않거나
	- 모든 서비스어카운트에 전체 권한을 부여했기 때문에 응답을 받음
* 위의 경우 서비스어카운트를 사용하는 유일한 이유
	- 마운트 가능한 시크릿을 적용하거나 서비스어카운트를 통해 이미지 풀 시크릿을 제공하는 것임
* 그러나 RBAC 인증 플러그인을 사용할 때는 추가 서비스어카운트를 만들어야 함

---
---
### 12.2 롤 기반 접근 제어 클러스터 보안
* 쿠버네티스 1.8.0 버전에서 RBAC 승인 플러그인은 GA(Generat Availability)로 승격히여 클러스터에서 기본적으로 활성화됨
	- 디폴트 서비스어카운트는 클러스터 상태를 볼 수 없으며 추가 권한을 부여히지 않는 한 어떤 식으로든 클러스터를 수정할 수 없음
```
참고 : RBAC 외에도 쿠버네티스에는 ABAC(Attribute-based Access Contr히）플러그인, 웹훅 플러그인, 
사용자 정의 플러그인 구현 같은 그 밖의 인증 플러그인이 포함돼 있다. 하지만 RBAC이 표준이다．
```

---
#### 12.2.1 RBAC 인증 플러그인 소개
* 쿠버네티스 API 서버를 설정하여 사용자가 요청한 작업이 수행할 수 있는지 승인 플러그인을 사용하여 확인 가능
	- API 서버는 REST 인터페이스를 노출하므로 사용자는 HTTP 요청을 서버로 전송해 작업을 수행
	- 사용자는 요청에 자격증명(인증 토큰, 사용자 이름 및 암호 또는 클라이언트 인증서)을 포함해 인증함

##### 액션
![HTTP 메소드 매핑](./Kubernetes in Action_12장_API서버보안/12장_01_HTTP메소드매핑하기.png)
* 전체 리소스 유형에 보안 권한 적용 가능
* RBAC 규칙은 리소스의 특정 인스턴스(Ex: myservice라는 서비스)에도 적용 가능
* URL 경로에도 권한을 줄 수 있음
```
참고 : 추가 동샤의 사용은 다음 장에서 설명하는 PodSecurityPolicy 리소스에 사용된다．
```

##### RBAC 플러그인
* 사용자의 역할을 보고 사용자가 액션을 수행할 수 있는지 여부를 결정할 때 핵심 요소로 사용함
* 주쳬는 하나 이상의 역할과 연관되며, 각 역할은 특정 리소스에서 특정 동사(verbs)를 수행할 수 있음
	- 주체 : 사람, 서비스어카운트 또는 사용자 또는 서비스어카운트의 그룹
* RBAC 플러그인을 통한 인증 과정
	- 네 가지 RBAC 관련 쿠버네티스 리소스를 만들어서 모든 인증을 수행할 수 있게 함

---
#### 12.2.2 RBAC 리소스 소개
* RBAC 인증 규칙은 네 가지 리조스로 구성되며 두 가지 그룹으로 그룹화 가능
	- 리소스에서 수행할 수 있는 동사를 지정하는 롤 및 클러스터롤
	- 위의 롤을 특정 사용자, 그룹 또는 서비스어카운트에 바인딩하는 롤바인딩 및 클러스터롤바인딩

![RBAC 리소스](./Kubernetes in Action_12장_API서버보안/12장_02_RBAC리소스.png)  

![RBAC 리소스 수준](./Kubernetes in Action_12장_API서버보안/12장_03_RBAC리소스수준.png)
* 네임스페이스 수준 리소스 : 롤과 롤바인딩
	- 단일 네임스페이스 공간에 롤과 롤바인딩이 여러 개 존재 가능
	- 롤바인딩이 네임스페이스 수준 리소스임에도 클러스터롤을 참조 가능
* 클러스터 수준의 리소스(넴임스페이스가 아닌) : 클러스터롤과 클러스터롤바인딩

##### 연습을 위한 환경 설정
* 쿠버네티스 버전 1.6 이상
* RBAC 플러스인 활성화
```
참고 : GKE 1.6 또는 1.7을 사용하는 경우 클러스터를 생성할 때 --no-enable-legacyauthorization
옵션을 사용해 레거시 승인을 명시적으로 비활성화해야 한다. 미니큐브를 사용하는 경우 미니큐브를 시작할 때
 --extra-config=apiserver.Authorization.Mode=RBAC 옵션을 줘서 RBAC을 활성화해야 할 수도 있다，
```
* 8장에서 RBAC를 비활성했으면, 활성화 하기
	- `$ kubectl delete clusterrolebinding permissive-binding`
* 네임스페이스 보안이 어떻게 작동하는지 확인하기 위해 다른 이름 공간에서 두 개의 포드를 실행
	- 단일 컨테이너 실행(kubectl-proxy 이미지 기반)
		+ 프록시는 인증 및 HTTPS를 처리하므로 API 서버 보안의 인증 측면에 집중 가능
	- curl 실행(kubectl exec를 사용)


##### 네임스페이스 생성 및 포드 실행
Ex) 다른 네임스페이스에서 테스트 포드 수행하기
```
$ kubectl create ns foo
namespace/foo created
$ kubectl run test --image=luksa/kubectl-proxy -n foo
pod/test created
$ kubectl create ns bar
namespace/bar created
$ kubectl run test --image=luksa/kubectl-proxy -n bar
pod/test created
``` 

Ex) 각 포드 내부에 쉘을 실행
```
$ kubectl get pod -n foo
NAME   READY   STATUS    RESTARTS   AGE
test   1/1     Running   0          106s

$ kubectl exec -it test -n foo -- sh
/ # 
``` 
* bar 네임스페이스의 포드도 다른 터미널에서 동일하게 수행

##### 포드에서 서비스 목록 나열
Ex) RBAC가 활성화 확인
```
/ # curl localhost:8001/apt/vl/namespaces/foo/services
...
  "status": "Failure",
  "message": "forbidden: User \"system:serviceaccount:bar:default\" cannot get path \"/apt/vl/namespaces/foo/services\"",
  "reason": "Forbidden",
  "details": {
  },
  "code": 403
```
* kubectl 프록시 프로세스가 리스닝하고 있는 localhost:8001에 연결 중
* 서비스어카운트의 디폴트 사용 권한으로 어떤 리소스든 리스트하거나 수정하는 것을 허용할 수 없음

---
#### 롤과 롤바인딩 사용
* 롤 리소스는 어떤 리소스에서 수행할 수 있는 액션을 정의함
	- 또는 RESTful 리소스에 수행할 수 있는 HTTP요청 유형

Ex) 룰의 정의(service-reader.yaml)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: foo			# 룰은 네임스페이스가 됨(생략되면, 현재 네임스페이스로 설정)
  name: service-reader
rules:
- apiGroups: [""]			# 서비스는 이름이 없는 core apiGroup의 리소스, ""로 표기함
  verbs: ["get", "list"]	# 개별 서비스를 가져오고, 모든 항목이 허용됨. 개별 서비스를 이름으로 get하고, 모든 항목을 list하는 것이 허용
  resources: ["services"]	# 이 룰/규칙은 서비스와 관련 있음(복수명을 사용해야함)
```
* 리소스를 지정할 때 복수형을 사용해야 함!
* 각 리소스 유형이 리소스 매니페스트의 apiVersion 필드(버전과 함께)데 지정하는 API 그룹에 속함
	- 롤 정의에서 정의에 포함된 각 규칙에 나열된 리소스에 apiGroup을 지정해야 함
	- 여러 API 그룹에 속한 리소스에 접근을 허용하는 경우 여러 규칙을 사용
```
참고 : 예를 들어 모든 서비스 리소스에 대한 접근을 허용하지만 추가 resourceNames 필드를 통해
이름을 지정해 특정 서비스 인스턴스에만 접근을 제한할 수도 있다．
```

![룰과 네임스페이스](./Kubernetes in Action_12장_API서버보안/12장_04_룰과네임스페이스.png)


##### 롤 생성
* `$kubectl create -f service-reader.yaml -n foo`
	- GKE를 사용하는 경우 클러스터 관리 권한이 없기에 명령을 실패할 수 있음
	- GKE에서 사용 `$ kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=user.email@address.com`
* YAML 파일로 롤읗 만드는 대신 `kubectl create role`이라는 명령어를 사용해 롤 생성 가능
	- `$ kubectl create role service-reader --verb=get --verb=list --resource=service -n bar`

##### 서비스어카운트에 롤을 바인딩
* 롤을 사용자, 서비스어카운트 또는 그룹(사용자 또는 서비스어카운트)이 될 수 있는 주쳬에 바인딩해야 의미 있음
* `kubectl create rolebinding test --role=service-reader --serviceaccount=foo:default -n foo`
```
참고 : 서비스어카운트 대신 사용자에게 롤을 바인딩하려면 --user 인수를 사용해 사용자 이름을 
지정하라. 그룹에 바인딩하려면 --group을 사용하라．
```

![룰과 룰바이딩](./Kubernetes in Action_12장_API서버보안/12장_05_룰과룰바이딩.png)

Ex) 롤을 참조하는 롤바인딩
```bash
$ kubectl get rolebinding test -n foo -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
...
  name: test
  namespace: foo
  resourceVersion: "376206"
  selfLink: /apis/rbac.authorization.k8s.io/v1/namespaces/foo/rolebindings/test
  uid: 9195605c-f01e-4fcb-b7ca-c16e2b94d9cb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role				# 롤바이딩은 service-reader 롤을 참조함
  name: service-reader
subjects:
- kind: ServiceAccount		# foo 네임스페이스상에서 기본 서비스어카운트에 바인드함
  name: default
  namespace: foo
```
* 롤바인딩은 항상 하나의 롤을 참조하지만, 롤을 여러 subjects에 바인딩할 수 있음


Ex) API 서버에서 서비스 가져오기
```bash
$ curl localhost:8001/api/v1/namespaces/foo/services
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/foo/services",
    "resourceVersion": "385370"
  },
  "items": []
}
```


---
---
### 12.3 요약
* 


---
## 출처
[^출처]: Kubernetes in Action-마르코 룩샤-에이콘


<!-- ![](./Kubernetes in Action_12장_API서버보안/) -->