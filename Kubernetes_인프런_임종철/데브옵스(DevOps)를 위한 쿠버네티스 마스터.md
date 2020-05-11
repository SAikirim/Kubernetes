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
	- 호스트 이름 etc /hostname
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
	- mkdir -p $HOME/.kube
	- sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	- sudo chown $(id -u):$(id -g) $HOME/.kube/config

* Pod Network 추가
	- kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
	- 이것을 잘해야 노드 추가 명령어가 잘 실행됨
	- root로 위에 명령어를 통해 'config'파일을 만들어야 실행됨

#### 슬레이브 노드 추가 (슬레이브 노드에서만 할 것)
* 앞서 설치한 대로 쿠버네티스 설치
* init 명령어 전까지만 수행 (init 명령어 실행하지 마세요)

* 이후 각 노드에서 관리자 권한으로 워커 노드를 참가 시킴 (콘솔에 출력된 메시지를 복붙)
	- sudo kubeadm join 10.0.2.15:6443 -token xwvbff.5xc67j8qc6ohl2it \
	-   --discovery-token-ca-cert-hash sha256:e19e9263aeb2340a602c2057966b71551e01a5e287d3f23b05073c7b248932e1

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
 [^출처]