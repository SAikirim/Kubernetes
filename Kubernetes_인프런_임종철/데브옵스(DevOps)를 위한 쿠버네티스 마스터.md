# 데브옵스(DevOps)를 위한 쿠버네티스 마스터


---
## 01. (쿠버네티스 들어가기 앞서) 왕초보도 따라하는 도커 기초

---
### 13.인프런 - 풀스택 워드프레스 컨테이너 이미지 만들기
* xampp 사용
	- apache, MariaDB, php가 설치되어 있음

Ex) xampp 설치
```zsh
$ docker run --name WP -p 80:80 -d tomsik68/xampp
Unable to find image 'tomsik68/xampp:latest' locally
~생략~
```
* tomsik68 이 만든 이미지
* `firefox http://127.0.0.1` 접속하여 확인 가능


Ex) 워드프레스 설치
```zsh
$ wget https://ko.wordpress.org/latest-ko_KR.tar.gz
$ tar xf latest-ko_KR.tar.gz
```
* 파일은 컨테이너에 넣어서 바로 실행하면 됨


Ex) 소유 권한 바꾸기 및 백업
```zsh
$ sudo docker exec -it WP bash
root@e9d20757fcfd:/# chown daemon. /opt/lampp/htdocs

root@e9d20757fcfd:/# cd /opt/lampp/htdocs
root@e9d20757fcfd:/opt/lampp/htdocs# mkdir backup
root@e9d20757fcfd:/opt/lampp/htdocs# mv * ./backup/
root@e9d20757fcfd:/opt/lampp/htdocs# exit
```
* 웹페이지에 쓰기가 가능해짐
* 웹 루트 : /opt/lampp/htdocs

Ex) 워드프레스 파일 복사
```zsh
$ docker cp wordpress WP:/opt/lampp/htdocs
$ sudo docker exec -it WP bash
root@e9d20757fcfd:/# chown daemon. /opt/lampp/htdocs
root@e9d20757fcfd:/# mv /opt/lampp/htdocs/wordpress/* /opt/lampp/htdocs/
root@e9d20757fcfd:/# exit
```
* `firefox http://127.0.0.1` 접속하여 확인 가능
	- 이후, 브라우저를 통해 진행함
	- `http://127.0.0.1/phpmyadmin/`로 접속
		+ `wordpress` DB 하나 만들기
* DB를 만들고, 설정해 계속 진행
	- 사이트 제목: wordpress_test
	- ID/PW: test/test1234

Ex) 재사용 및 백업
```zsh
$ sudo docker stop WP

$ docker commit WP kyj9724/wordpress
sha256:207a2dd75822dbef0fe22d517baab7074985f1cbbdd3828dbca014da195ed6df

$ docker login 

$ docker push kyj9724/wordpress
```

Ex) 확인
```zsh
$ docker rm `docker ps -a -q`
$ docker run -d -p 80:80 --rm kyj9724/wordpress
43cca2e197590623228449ee0445d4a3af5e09a67fad0fd34caab02bbadde74b
```
* `docker restart 43cca2e` 안되면 재시작

---
## 02.쿠버네티스 들어가기

---
### 0. 쿠버네티스 소개 


#### 쿠버네티스 시작
* 오랜 세월 동안 구글은 보그 (Borg)라는 내부 시스템을 개발
* 애플리케이션 개발자와 시스템 관리자가 수천 개의 애플리케이션과 서비스를 관리하는 데 도움
* 조직 규모가 클 때 엄청난 가치를 발휘
* 수십만 대의 시스템을 가동할 때 사용률이 조금만 향상돼도 수백만 달러의 비용 절감 효과
* 구글은 보그와 오메가를 15년 동안 비밀로 유지
* 2014 년 구글 시스템을 통해 얻은 경험을 바탕으로 한 오픈소스 시스템인 쿠버네티스를 출시
	- GO언어를 바탕으로 만듬

#### 인프라의 추상화
* 많은 노드(컴퓨터)들을 하나의 거대한 컴퓨터로 인식
	- 클러스터링
* 컨테이너 애플리케이션을 쉽게 배포 관리하도록 돕는 소프트웨어 시스템

#### 쿠버네티스의 장점
* 애플리케이션 배포 단순화
	- 특정 베어메탈을 필요로 하는 경우 (예:SSD/HDD)
	- 라벨링을 통한 쉬운 배포 가능

* 하드웨어 활용도 극대화
	- 클러스터의 주변에 자유롭게 이동하여 실행중인 다양한 애플리케이션 구성 요소를 클러스터 노드의 가용 리소스에 최대한 맞춰 서로
섞고 매치
	- 노드의 하드웨어 리소스를 최상으로 활용
	
* 상태 확인 및 자가 치유
	- 애플리케이션 구성 요소와 실행되는 노드를 모니터링하고 노드 장애 발생시 다른 노드로 일정을 자동으로 재조정
	- 운영자는 정규 근무 시간에만 장애가 발생한 노드를 처리(일이 편해진다)
	
* 오토스케일링
	- 개별 애플리케이션의 부하를 지속적으로 모니터링할 필요 없이
	- 자동으로 리소스를 모니터링하고 각 애플리케이션에서 실행되는 인스턴스 수를 계속 조정하도록 지시 가능
	
* 애플리케이션 개발 단순화
	- 버그 발견 및 수정(완전히 개발환경과 같은 환경을 제공하기 때문)
	- 새로운 버전 출시 시 자동으로 테스트, 이상 발견 시 롤 아웃

#### 쿠버네티스의 궁극의 목적
* 개발자 돕기 : 핵심 애플리케이션 기능에 집중
	- 애플리케이션 개발자가 특정 인프라 관련 서비스를 애플리케이션에 구현하지 않아도 됨
		+ 쿠버네티스에 의존해 서비스 제공
	- 서비스 검색 확장 로드 밸런싱 자가 치유 리더 선출 등
	- 애플리케이션 개발자는 애플리케이션의 실제 기능을 구현하는 데 주력
	- 인프라와 인프라를 통합하는 방법을 파악하는데 시간을 낭비할 필요 없음
* 운영 팀 돕기 : 이 효과적으로 리소스를 활용
	- 실행을 유지하고 서로 통신할 수 있도록 컴포넌트에 정보를 제공
	- 애플리케이션이 어떤 노드에서 실행되는 상관 없음 (신경 쓰지 않아도 됨)
	- 언제든지 애플리케이션을 재배치 가능
	- 애플리케이션을 혼합하고 매칭시킴으로써 리소스를 매칭
* DevOps 극대화
		+ 컨테이너 -> 오케스트라 툴

---
### 1. 아키텍처 

#### 쿠버네티스 클러스터 아키텍처
* 쿠버네티스의 클러스터는 하드웨어 수준에서 많은 노드로 구성되며 두 가지 유형 나뉨
	- 마스터 노드: 전체 쿠버네티스 시스템을 관리하고 통제하는 쿠버네티스 컨트롤 플레인을 관장
	- 워커 노드: 실제 배포하고자 하는 애플리케이션의 실행을 담당

### 컨트롤 플레인(마스터)
* 컨트롤 플레인에서는 클러스터를 관리하는 기능
* 단일 마스터 노드에서 실행하거나 여러 노드로 분할되고 복제돼 고가용성을 보장
* 클러스터의 상태를 유지하고 제어하지만 애플리케이션을 실행하지 않음


---
### 2. 클러스터 설치 브리핑
* Master, Worker1, Worker2로 구성 실습
* 모두 docker 설치는 필수

#### 쿠버네티스 설치 필요 사항
* 버추얼박스에서 각 노드에서 복제하면서 반드시 변경해야 하는 설정
	- 호스트 이름 /etc/hostname
	- 네트워크 인터페이스 변경
	- NAT 네트워크 설정 (NAT랑 다름)
	- (호스트 이름 변경하려면 반드시 리붓)
* hostname으로 노드를 구별함(중요)
* MAC주소 및 NAT 설정
	- 통신이 되게 만듦

#### kubernetes를 관리하는 명령어
* kubeadm
	- 클러스터를 부트스트랩하는 명령
	- 인증성 관리, 마스터 노드를 만들 수 있도록 초기화 함, 쿠버네티스 설치
* kubelet
	- 클러스터의 모든 시스템에서 실행되는 구성 요소로 창 및 컨테이너 시작과 같은 작업을 수행
* kubectl
	- 커맨드 라인 util 은 당신의 클러스터와 대화
	- 클라이언트 프로그램

#### 쿠버네티스 우분투에 설치
* 다음 내용을 install.sh 파일에 작성하고 chmod로 권한을 주고 실행
* 쿠버네티스 설치 사이트 아래 스크립트 내용이 있음
	- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

* 신뢰할 수 있는 APT 키를 추가
	- apt-get update && apt get install -y apt-transport-https curl
	- curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

* 그리고 아래의 명령어로 Repository 를 추가하고 Kubernetes 를 설치
	```bash
	cat <<EOF> /etc/apt/sources.list.d/kubernetes.list
	deb https://apt.kubernetes.io/kubernetes-xenial main
	EOF
	apt-get update
	apt-get install -y kubelet kubeadm kubectl
	apt-mark hold kubelet kubeadm kubectl
	```

#### Master 노드 초기화(마스터 노드에서만 할 것)
* Master노드를 초기화를 가장 먼저 수행 (사용할 포드 네트워크 대역을 설정)
	- `sudo kubeadm init`
	- 오류/타임아웃 발생시 '신뢰할 수 있는 APT 키를 추가' 실시 
* 스왑 에러 발생 시 스왑 기능 제거
	- `sudo swapoff -a` : 현재 커널에서 스왑 기능 끄기
	- `sudo sed -i '/swap / s/^\(.*\)$/#\1/g' /etc/fstab` : 리붓 후에도 스왑 기능 유지
	- __워커노드에서도 스왑 기능 제거 필요!!!__

* Kubernetes에서 스왑을 비활성화하는 이유
	- Kubernetes 1.8 이후, 노드에서 스왑을 비활성화해야 함 (또는 fail swap on을 false 로 설정)
	- kubernetes의 아이디어는 인스턴스를 최대한 100에 가깝게 성능을 발휘하는 것
	- 모든 배포는 CPU/ 메모리 제한을 고정하는 것이 필요
	- 따라서 스케줄러가 포드를 머신에 보내면 스왑을 사용하지 않는 것이 필요
	- 스왑 발생시 속도가 느려지는 이슈 발생
	- 성능을 위한 것

[참고문헌] https://serverfault.com/questions/881517/why-disable-swap-on-kubernetes

#### 클러스터를 사용 초기 세팅(마스터 노드에서만 할 것)
* 다음을 일반 사용자 계정으로 실행 콘솔에 출력된 메시지를 복붙
	```
	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config
	```

* Pod Network 추가
	`kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`
	- 이것을 잘해야 노드 추가 명령어가 잘 실행됨
	- root로 위에 명령어를 통해 'config'파일을 만들어야 실행됨

#### 슬레이브 노드 추가 (슬레이브 노드에서만 할 것)
* 앞서 설치한 대로 쿠버네티스 설치
* init 명령어 전까지만 수행 (init 명령어 실행하지 마세요)

* 이후 각 노드에서 관리자 권한으로 워커 노드를 참가 시킴 (콘솔에 출력된 메시지를 복붙)
	```
	sudo kubeadm join 10.0.2.15:6443 -token xwvbff.5xc67j8qc6ohl2it \
	   --discovery-token-ca-cert-hash sha256:e19e9263aeb2340a602c2057966b71551e01a5e287d3f23b05073c7b248932e1
	```

#### Error 발생시
[참고] https://xoit.tistory.com/94

Ex) 오류
```bash
 [ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1
```
* 해결
	- echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
	- kubeadm init(마스터에서 발생시 사용)

#### 마지막으로 연결된 노드들의 상태 확인
* kubectl get nodes
	- STATUS 값이 NotReady 상태인 경우, Pod Network 가 아직 deploy 되기 전일 수 있음
	- 장시간 기다려도 변경되지 않으면 앞에서 설정한 “Pod Network 추가” 과정이 잘못 됐을 수 있음
	- [참고] https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

##### 일반적인 사용자와 마스터 노드, 워커 노드 연결 관계
![일반적인 상황](./02.쿠버네티스 들어가기/2. 클러스터 설치 브리핑_00.png)

##### 우리가 설정한 환경에서 마스터 노드와 슬레이브 노드 연결 관계
![설정한 상황](./02.쿠버네티스 들어가기/2. 클러스터 설치 브리핑_01.png)

#### 재설치시 유의상황
* Kubeadm init 또는 join 명령을 실행할 때 중복된 실행으로 문제가 생기는 경우 kubeadm reset명령을 통해서 초기화한다


---
### 7. GKE를 활용한 쿠버네티스 사용

#### Google cloud 의 관리형 kubernetes 서비스인 Google Kubernetes Engine(GKE)
* GKE 는 Kubernetes 를 쉽게 사용자가 활용할 수 있도록 관리형으로 제공
* 규모에 맞춘 컨테이너식 애플리케이션 관리
* 다양한 애플리케이션 배포
* 고가용성을 통한 원활한 운영
* 수요에 맞게 간편하게 확장
* Google 네트워크에서의 안전한 실행
* 온프레미스 및 클라우드 간의 자유로운 이동

#### GCP 가입 및 가입오류 해결
1.회원 가입에서 다음이 눌리지 않는 경우 해결 방법
	- https://private-space-tistory.com/41에서 자세히 설명
	- 요약: https ://cloud.google.com/gcp/getting-started/?hl=ko를 접속해서 "콘솔"로 접속하면 가입 오류 없이 회원 가입 가능
2. 회원가입 시 카드 정보 오류가 발생하는 경우
	- 회원가입 시 카드 정보 오류가 발생하는 경우 모바일로 접속해서 등록하면 회원 가입 가능한 경우가 있음
3 사용할 수 없는 결제 정보인 경우
	- 다른 ID 를 새로 생성하면 해결되는 경우가 있음

#### GCP 프로젝트 생성시 유의사항
* 프로젝트를 만들때 '위치:회사 조직'을 설정하면 권한이 넘어가 프로젝트 삭제를 못함, 기본값을 사용
* 기본적으로 기능을 사용할 때 설치됨, GKE를 선택하면 기능 활성화하는데 1분 정도 걸림
* 클러스터: 마스터 버전 선택시, 구글이 쿠버네티스를 커스텀마이즈하여 사용하기 때문에 업데이트에 시간이 걸림, 기본값 사용 권장
* GCP 장점: Azure와 다르게 인스턴스 비용 지불 없이, 클러스터에 접속 가능(Google Cloud Shell)

---
### 9. AWS EKS를 활용한 쿠버네티스 사용
* GCP와 다르게 프로젝트를 삭제해도, 다른 리소스들이 남음
	- 일일히 삭제가 필요
* 유로 서비스이기 때문에 실습없음, 프리티어에 12개월 동안 액세스 가능

#### Amazon Elastic Container Service for Kubernetes(Amazon EKS)
* AWS에서 Kubernetes를 손쉽게 실행하도록 하는 관리형 서비스
* 여러 가용 영역에서 Kubernetes 제어 플레인 인스터스를 실행하여 고가용성을 보장
* 비정상 제어 플레인 인스턴스를 자동으로 감지하고 교체
* 자동화된 버전 업그레이드를 제공
* 여러 AWS 서비스와 통합되어 다음을 포함한 애플리케이션에 대한 확장성과 보안을 제공
	- 컨테이너 이미지용 Amazon ECR
	- 로드 배포용 Elastic Load Balancing
	- 인증용 IAM
	- 격리용 Amazon VPC
	
#### 아마존 EKS 시작하기
* https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/getting-started.html
* AWS 에서는 EKS 를 시작하는 두 가지 방법을 제공
	- eksctl 로 시작하기
		+ Amazon EKS 를 시작하는 가장 빠르고 쉬운 방법
		+ 클러스터를 생성 및 관리하기 위한 간단한 명령줄 유틸리티인 eksctl 제공
		+ 필요한 모든 리소스를 설치
		+ kubectl 명령 줄 유틸리티
		+ 자격증명이 먼저 필요, Windows에선 에러 발생 가능있음
	- AWS Management
		+ AWS Management 콘솔 사용
		+ Amazon EKS를 시작할 때 필요한 모든 리소스를 생성 가능
		+ Amazon EKS 또는 AWS CloudFormation 콘솔을 사용하여 각 리소스를 수동으로 생성
		+ 각 리소스의 생성 방법 및 리소스 간의 상호 작용을 완벽하게 파악 가능
		+ Amazon EKS를 시작하는 방법으로는 더 복잡하고 시간도 많이 걸림

---
### 10. 쿠버네티스에서 앱 실행해보기

Ex) main.go
```go
package main

import(
        "fmt"
        "github.com/julienschmidt/httprouter"
        "net/http"
        "log"
        "os"
)

func Index(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
        hostname, err := os.Hostname()
        if err == nil {
                fmt.Fprint(w, "Welcome! " + hostname +"\n")
        } else {
                fmt.Fprint(w, "Welcome! ERROR\n")
        }
}

func main() {
        router := httprouter.New()
        router.GET("/", Index)

        log.Fatal(http.ListenAndServe(":8080", router))
}
```
* Google Cloud Shell 또는 로컬 가상 머신에서 실습

#### Go 언어 설치 및 프로그램 빌드
* apt install golnag 를 사용해서 고 랭기지를 설치
* go get 명령을 사용해 외부 라이브러리 임포트
* go build 명령으로 main go 를 빌드하여 main 파일 생성
* main 명령 실행 후 정상 동작하는 지 테스트

```bash
apt install golang
go get github.com/julienschmidt/httprouter
go build main.go
main
```
#### dockerfile 작성
* FROM golang 1.11 버전의 컨테이너 이미지를 사용
* 로컬 디렉터리의 main 파일을 이미지의 /usr/src/app 디렉터리에 동일한 이름으로 추가
* 이미지를 실행할 때 /usr/src/app/main 실행

Ex) Dockerfile
```zsh
Ubuntu :: ~/http_go # cat Dockerffile
FROM golang:1.11
WORKDIR /usr/src/app
COPY main /usr/src/app
CMD ["/usr/src/app/main"]
```
* 작성 후, 빌드 : docker build -t http-go .
* 실행 후, 확인 : docker run -d -p 8080:8080 --rm http-go

### 컨테이너 푸시하기
```bash
docker tag http-go gasbugs/http-go
(docker login)
docker push gasbugs/http-go
```

---
#### 처음으로 만드는 쿠버네티스 앱
* 보통 배포하려는 모든 컴포넌트의 설명이 기술된 JSON 또는 YAML 매니페스트를 준비 필요
* 이를 위해서는 쿠버네티스에서 사용되는 컴포넌트 유형을 잘 알아야 함
* 여기서는 명령어에 몇 가지 옵션으로 디스크립션을 간단히 전달하여 한 줄로 앱을 실행
`$ kubectl create deploy http-go --image=gasbugs/http-go`
`kubectl expose  deployment http-go --name http-go-svc --port=8080`
`kubectl get svc -w`

---
#### 포드란?
* 포드 하나에는 프로세스(컨테이너)하나를 권장

#### 웹 애플리케이션 만들어보기
* 실행 중인 포드는 클러스터의 가상 네트워크에 포함돼 있음
* 어떻게 액세스 할 수 있을까
* 외부에서 액세스하려면 서비스 객체를 통해 IP 를 노출하는 것이 필요
* LoadBalancer 라는 서비스를 작성하면 외부 로드 밸런서가 생성
* 로드 밸런서의 공인 IP를 통해 포드에 연결 가능
	- 하지만 로컬 쿠버네티스에서는 동작하지 않으며 externalDNS 가 필요함,
	- 이 기능은 GKE, EKS 같은 클라우드에서 사용 가능(구글 , AWS 계정 필요)

#### 디플로이먼트의 역할
* 디플로이먼트는 레플리카셋을 생성
* 레플리카셋은 수를 지정하여 알려주면 그 수만큼 포드를 유지
* 어떤 이유로든 포드가 사라지면 레플리카셋은 누락된 포드를 대체할 새로운 포드를 생성

#### 서비스의 역할
* 포드는 일시적이므로 언제든지 사라질 가능성 존재
* 포드가 다시 시작되는 경우에는 언제든 IP와 ID 변경됨
* 서비스는 변화하는 포드 IP 주소의 문제를 해결하고 단일 IP 및 포트 쌍에서 여러 개의 포드 노출
* 서비스가 생성되면 정적 IP 를 얻게 되고 서비스의 수명 내에서는 변하지 않음
* 클라이언트는 포드에 직접 연결하는 대신 IP 주소를 통해 서비스에 연결
* 서비스는 포드 중 하나로 연결을 포워딩

#### 서비스 상세
* 포드의 통신을 관리
	- 기본적으로 서비스는 로드밸런스 기능을 가짐
	- 내부통신(web->db: 코드에 링크로 연결(도커), 이름으로 통신(IP가 가변적이라 사용)), 외부 통신 가능
	- 외부 : '노드'들의 ip:port 통신
	- 내부 : '포드' 내 통신
	- 로드밸러서와 노드ip 대역은 같음
* 서비스없이 포드끼리 IP통신은 가능하나, IP는 가변적이라 서비스가 관리
* 서비스는 포드와 연결

### !!쿠버네티스 네트워크를 나누는 이유!!
* '트래픽'을 나누기(분산하기) 위해
	- pod끼리 통신, 노드끼리 통신, 서비스끼리 통신

#### 애플리케이션의 수평 스케일링
* 쿠버네티스를 사용해 얻을 수 있는 큰 이점 중 하나는 간단하게 컨테이너의 확장이 가능하다는 점
* 포드의 개수를 늘리는 것도 쉽게 가능
* 포드는 디플로이먼트가 관리
`$ kubectl scale deploy http-go --replicas=3`

#### 직접 앱에 접근하기
* curl 명령어로 요청
* external IP 를 할당 받지 못했기 때문에 포드의 힘을 빌려 요청
`$ kubectl exec http-go-XXXXXX-bt4xq -- curl -s http://10.109.140.155:8080`
	- 포드를 변경하면서 시도
	- '--' : kubectl의 명령행 옵션의 끝을 의미
		+ 더블 대시를 쓰지 않는다면 -S 옵션은 kubectl exec의 옵션으로 해석돼서 다음과 같이 이상한 오류가 발생할


---
 [^출처]