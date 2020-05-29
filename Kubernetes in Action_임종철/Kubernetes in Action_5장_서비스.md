# Kubernetes in Action

---
## 5장 볼륨：클라이언트가 포드를 검색하고 통신을 가능하게 함

---
### 5.1 서비스 소개
![서비스 기본 구조](./Kubernetes in Action_5장_서비스/5장_00_서비스기본구조.png)

#### 5.1.1 서비스 생성

##### kubecti expose를 통한 서비스 생성
Ex) kubia-svc.yanil
```bash
kubectl expose deployment hello-world --type=LoadBalancer --name=my-service
```

##### YAML 디스크립터를 통한 서비스 생성

Ex) 서비스의 정의(kubia-svc.yaml)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia

spec:
  ports:
    - port: 80 			# 서비스가 사용할 포트
      targetPort: 8080	# 서비스가 포워드할 컨테이너 포트
  selector:				# 라벨이 app=kubia인 모든 포드는 이 서비스에 속한다.
    app: kubia
```
* 80 port로 들어오는 연결을 허용하고 각 연결을 대상으로 app=kubta 라벨 셀렉(label selector)에 매칭되는 포드 중 하나를 8080 포트로 라우팅함

##### 클러스터 안에서 서비스 테스트
* 확실한 방법은 서비스의 클러스터 IP주소로 요청을 보내고 응답을 로그로 남기는 포드를 생성하는 것
* 노드 중 하나로 ssh 접속을 수행 하고 curl 명령을 사용
* Kubectl exec 명령어를 통해 이미 존재하는 포드들 중 하나에서 curl 명령을 수행

##### 원격으로 실행 중인 컨테이너에 명령에 실행
```bash
$ kubectl exec kubia-7nog1 -- curl -s http://10.11.249.153
```

#### 동일한 서비스에서 여러 개의 포트 노출
Ex) 서비스의 정의(다중 포트 설정하기)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia

spec:
  ports:
    - name: http
	  port: 80 			# 서비스가 사용할 포트
      targetPort: 8080	# 서비스가 포워드할 컨테이너 포트
    - name: https
	  port: 443
	  targetPort: 8443
  selector:				# 라벨이 app=kubia인 모든 포드는 이 서비스에 속한다.
    app: kubia
```

#### 이름이 지정된 포트 사용
Ex) 포드 정의에 포트 이름 설정하기
```yaml
kind: pod
spec:
  containers:
    - name: kubia
	  ports:
	    - name: http			# 컨테이너 포트 8080은 http 이름으로 설정한다.
		  containerPort: 8080
		- name: https
		  containerPort: 8443
```

Ex) 서비스에 이름 지정된 포트 참조하기
```yaml
apiVersion: v1
kind: Service

spec:
  ports:
    - name: http
	  port: 80 			# 포트 80은 http라 불리는 컨테이너 port에 매핑된다.
      targetPort: http
    - name: https
	  port: 443
	  targetPort: https
```
* 포트 이름을 지정하는 것의 가장 큰 이점은 서비스 스펙의 변경 없이 포트 번호를 변경할 수 있음
	- 포트가 현재 http라는 이름으로 8080 포트를 사용하더라도 후에 필요에 의해 포트를 80으로 변경하는 것이 가능


#### 5.1.2 서비스 검색

##### 환경 변수를 이용한 서비스 검색
Ex) 컨테이너 내의 서비스와 연관된 환경 변수
```
$ kubectl exec kubta-3tn1y env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=kubia-3in1v
KUBERNETES_SERVICE_HOST=10.111.240.1
KUBERNETES_SERVICE_PORT=443
...
KUBIA_SERVICE_HOST=10.111.249.153 	# 서비스의 클러스터 IP
KUBIA_SERVICE_PORT=80				# 서비스가 사용 가능한 포트
...
```

##### DNS를 이용한 서비스 검색
* kube-dns가 DNS 서버를 실행함
	- 이 DNS서버는 클러스터에서 실행하는 다른 모든 포드가 자동적으로 사용하도록 구성됨
* 각 서비스는 내부 DNS 서버에서 DNS 항목을 가져옴
	- 서비스 이름을 알고 있는 클라이언트 포드는 환경 변수를 사용하는 대신 FQDN(정규화된 도메인 이름)을 통해 액세스할 수 있음

##### FQDN을 이용한 서비스 연결
아래의 FQDN을 통해 백엔드 데이터베이스 서비스에 연결할 수 있음
```bash
backend-database.default.svc.cluster.local
```
* backend-database : 서비스 이름 
* default : 서비스가 정의된 네임스페이스
* svc.cluster.local : 모든 클러스터의 로컬 서비스 이름
* 서비스로 연결할 경우
	- 생략이 가능해
	- `backend-database`만으로 서비스를 지칭 가능

---
### 5.2 클러스터 외부 서비스에 연결

#### 5.2.1 서비스 엔드포인트 소개 
Ex) 엔드포인트 확인
```bash
$ kubecti get endpotnts kubta
NAME    ENDPOINTS                                      AGE
kubia   10.36.0.1:8080,10.44.0.1:8080,10.47.0.1:8080   100m
```

#### 5.2.2 수동으로 서비스 엔드포인트 설정
* 서비스의 엔드포인트를 서비스와 분리하면 엔드포인트를 수동으로 구성하고 업데이트할 수 있음

##### 셀렉터 없이 서비스 생성
Ex) 포드 셀렉터 없는 서비스(external-service.yaml)
```
apiVersion: v1
kind: Service
metadata:
  name: external-service	# 엔드포인트 객체의 이름과 매칭되야 함
spec:						# 서비스는 셀렉터를 따로 정의하고 있지 않음
  ports:
    - port: 80
```
* 80 포트로 들어오는 연결을 처리하는 'external-service'라는 서비스를 정의
	- 이 서비스를 위해 포드 셀렉터를 정의하지는 않음

##### 셀렉터 없이 서비스를 위한 엔드포인트 리소스 생성
Ex) 직접 생성한 엔드포인트 리소스(external-service-endpoints.yaml)
```
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service	# 엔드포인트 객체의 이름, 서비스의 이름과 매칭돼야 함
subset:						
  - addresses
    - ip: 11.11.11.11		# 서비스가 연결을 포워딩할 엔드포인트의 IP들
	- ip: 22.22.22.22
	port:
	  - port:80				# 엔드포인트 대상 포트
```
* 서비스를 위한 IP와 포트 정보 필요

![서비스, 포드](./Kubernetes in Action_5장_서비스/5장_01.png)

#### 5.2.3 외부 서비스를 위한 별칭 생성

##### ExternalName 서비스 생성
Ex) ExternalName 타입의 서비스(external-service-externalname.yml
```
apiVersion: v1
kind: Service
metadata:
  name: external-service	
spec:						
  type: ExternalName
  externalName: someapi.somecompany.com	# 실제 서비스의 전체 도메인 주소(FQDN, Fully Qualified Domain Name)
  port:
    - port:80				
```
* Exte rnalNarne 서비스는 DNS 레벨에서만 구현됨
	- 간단히 CNAME DNS 레코드는 서비스를 위해 생성
	- 서비스로 연결하는 클라이언트는 서비스 프록시를 완전히 통하지 않고 직접 외부서비스로 연결
	- 이런 이유로 서비스를 위한 이런 타입은 클러스터 IP를 얻지 못함

###### 참고
* CNAME 레코드는 숫자 형식의 IP 주소 대신에 FQDN(fully qualified domain name）을 지칭함


---
### 5.3 외부 서비스에서 외부 클라이언트로
* 서비스가 외부에서 액세스 가능한 방법
	- NodePort 서비스 탸입으로 설정하기
		+ NodePort 서비스의 각 클러스터 노드는 노드 자체의 이름을 통해 포트를 열고 포트에서 발생한 트래픽을 서비스로 리다이렉트 함
	
	- LoadBalancer 서비스 타입으로 설정하기（NodePort 타입의 확장형）
		+ 쿠버네티스가 실행 중인 클라우드 인프라스트럭처에 프로비전(provision)된 지정된 LoadBalancer를 통해 서비스 액세스가능하게 됨
		+ LoadBalancer는 발생한 트래픽을 모든 노드에서 노드포트로 리다이렉트 함
		+ 클라이언트는 로드 밸런서 IP를 통해 서비스에 접속함
		
	- 하나의 IP주소를 통해 여러 서비스를 제공하는 근본적으로 다른 매커니즘인 인그레스 리소스 생성하기
		+ HTTP 레벨（네트워크 7계층）수준에서 동작하기 때문에 4계층 서비스보다 좀 더 기능을 제공함
		
		
		
#### 5.3.1 NodePort 서비스 사용
* NodePort
	- 서비스를 생성해 쿠버네티스가 모든 노드를 대상으로 포트를 예약함
	- 모든 노드에 걸쳐 동일한 포트 번호를 사용하게 됨

##### NodePort 서비스 생성
Ex) NodePort 서비스 정의(kubia-svc-nodeport.yaml)
```
apiVersion: v1
kind: Service
metadata:
  name: kubta-nodeport
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30123
  selector:
    app: kubia		
```
![NodePort](./Kubernetes in Action_5장_서비스/5장_02_NodePort.png)


#### 5.3.2 외부 로드 밸런서를 이용한 서비스 노출
* 로드 밸런서는 자신만의 고유하면서 외부에서 액세스가 가능한 IP 주소를 갖고 모든 연결을 서비스로 리다이렉트함
* 로드 밸런서의 IP 주소를 통해 서비스에 액세스할 수 있음
* LoadBalancer 서비스를 지원하지 않는 환경에서 실행하면
	- 밸런서는 프로비전되지 않을 것임
	- LoadBalancer 서비스는 NodePort 서비스의 확장이기 때문에 서비스는 NodePort 서비스처럼 동작할 것임

##### LoadBalancer 서비스 생성
Ex) LoadBalancer 타입 서비스 정의(kubia-svc-.yaml)
```
apiVersion: v1
kind: Service
metadata:
  name: kubta-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia		
```
![LoadBalancer](./Kubernetes in Action_5장_서비스/5장_03_로드밸런서.png)

#### 5.3.3 외부 연결의 특성

---
### 5.4 인그레스 리소스를 이용해 외부로 서비스 노출하기

#### 5.4.1 인그레스 리소스 생성
#### 5.4.2 인그레스를 이용한 서비스 액세스
#### 5.4.3 하나의 인그레스로 다수의 서비스 노출
#### 5.4.4 TLS 트래픽을 처리하기’ 위한 인그레스 설정

---
### 5.5 포드가 연결을 수락할 준비가 됐을 때 신호 보내기

#### 5.5.1 레디네스 프로브 소개
#### 5.5.2 포드로 레디네스 프로브 추가
#### 5.5.3 실제 환경에서 레디네스 프로브의 역할

---
### 5.6 헤드리스 서비스 사용해 개별 포드 찾기

#### 5.6.1 헤드리스 서비스의 생성
#### 5.6.2 DNS를 통해 포드 찾기
#### 5.6.3 모든 포드 찾기（준비되지 않은 포드까지)

---
### 5.7 서비스의 문제 해결

---
### 5.8 요약


![서비스 기본 구조](./Kubernetes in Action_5장_서비스/5장_00_서비스기본구조.png)
---
## 출처
[^출처]: Kubernetes in Action-마르코 룩샤-에이콘
