# Kubernetes in Action


---
## 6장 볼륨： 컨테이너에 디스크 스토리지 연결

---
### 6.2 볼륨을 사용한 컨테이너 사이의 데이터 공유(emptyDir 볼륨)

---
### 6.3 워커노드 파일 시스템에 액세스하기(hostPath 볼륨)


---
### 6.4 영구 스토리지 사용

#### 6.4.1 포드 볼륨에서 GCE 영구 디스크 사용


---
### 6.7 PersistentVolume and PersistentVolumeClaim[^조대협의 블로그]
* 일반적으로 디스크 볼륨을 설정하려면 물리적 디스크를 생성해야 함
* 쿠버네티스는 인프라에 대한 복잡성을 추상화를 통해서 간단하게 하고, 개발자들이 손쉽게 필요한 인프라(컨테이너,디스크, 네트워크)를 설정할 수 있도록 하는 개념을 가지고 있음
* 인프라에 종속적인 부분은 시스템 관리자가 설정하도록 하고, 개발자는 이에 대한 이해 없이 간단하게 사용할 수 있도록 구현함
	- 디스크 볼륨 부분에 PersistentVolumeClaim(이하 PVC)와 PersistentVolume(이하 PV)라는 개념을 도입

* 시스템 관리자가 실제 물리 디스크를 생성한 후
	- 이 디스크를 PersistentVolume이라는 이름으로 쿠버네티스에 등록
* 개발자는 Pod를 생성할때, 볼륨을 정의하고
	- 이 볼륨 정의 부분에 물리적 디스크에 대한 특성을 정의하는 것이 아니라 __PVC__를 지정
	- 지정한 PVC를 관리자가 생성한 PV와 연결

![PVC 및 PV구조](./Kubernetes in Action_6장_볼륨/6장_00_PVC및PV구조.png)
* __주의할점__
	- 볼륨은 생성된후에, 직접 삭제하지 않으면 삭제되지 않음
	- PV의 생명 주기는 쿠버네티스 클러스터에 의해서 관리되면 Pod의 생성 또는 삭제에 상관없이 별도로 관리됨
		+ Pod와 상관없이 직접 생성하고 삭제해야 함

#### PersistentVolume
Ex) NFS 파일 시스템 5G를 pv0003이라는 이름으로 정의
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003

spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  accessModes:
    - ReadWriteOnce
  storageClassName: slow
  mountOptions:
    - hard
	- nfsvers=4.1
  nfs:
    path: /tmp
	server: 172.17.0.2
```
* Capacity
	- 볼륨의 용량을 정의
	- 현재는 storage 항목을 통해서 용량만을 지정하는데, 향후에는 필요한 IOPS나 Throughput등을 지원할 예정임 
* VolumeMode 
	- VolumeMode는 Filesystem(default) 또는 raw를 설정할 수 있음
	- 볼륨이 일반 파일 시스템인데, raw 볼륨인지를 정의
* Reclaim Policy
	- PV는 연결된 PVC가 삭제된 후, 다시 다른 PVC에 의해서 재사용이 가능
	- 재사용시에 디스크의 내용을 지울지 유지할지에 대한 정책을 Reclaim Policy를 이용하여 설정이 가능
		+ Retain : 삭제하지 않고 PV의 내용을 유지함
		+ Recycle : 재사용이 가능하며, 재사용시에는 데이타의 내용을 자동으로 rm -rf 로 삭제한 후 재사용함
		+ Delete : 볼륨의 사용이 끝나면, 해당 볼륨은 삭제함. AWS EBS, GCE PD,Azure Disk등이 이에 해당함
	- __Reclaim Policy은 모든 디스크에 적용이 가능한것이 아니라, 디스크의 특성에 따라서 적용이 가능한 Policy가 있고, 적용이 불가능한 Policy가 있음__
* AccessMode
	- AccessMode는 PV에 대한 동시에 Pod에서 접근할 수 있는 정책을 정의함 
		+ ReadWriteOnce(RWO) : 해당 PV는 하나의 Pod에만 마운트되고 하나의 Pod에서만 읽고 쓰기가 가능함
		+ ReadOnlyMany(ROX) : 여러개의 Pod에 마운트가 가능하며, 여러개의 Pod에서 동시에 읽기가 가능함. 쓰기는 불가능
		+ ReadWriteMany(RWX) : 여러개의 Pod에 마운트가 가능하고, 동시에 여러개의 Pod에서 읽기와 쓰기가 가능
	- __위와 같이 여러개의 모드가 있지만, 모든 디스크에 사용이 가능한것은 아니고 디스크의 특성에 따라서 선택적으로 지원됨__

##### PV의 라이프싸이클
* PV는 생성이 되면, Available 상태가 됨
* 이 상태에서 PVC에 바인딩이 되면 Bound 상태로 바뀌고 사용이 되며,
	- 바인딩된 PVC가 삭제 되면, PV가 삭제되는 것이 아니라  Released 상태가 됨
	- Available이 아니면 사용은 불가능하고 보관 상태가 됨

##### PV 생성 (Provisioning)
* PV의 생성은 앞에서 봤던것 처럼 yaml 파일 등을 이용하여, 수동으로 생성을 할수도 있음
* 또는 설정에 따라서 필요시마다 자동으로 생성할 수 있게 할 수 있음
	- 이를 Dynamic Provisioning(동적 생성)이라고 함 

#### PersistentVolumeClaim
*PVC는 Pod의 볼륨과 PVC를 연결(바인딩/Bind)하는 관계 선언

Ex)PVC의 예제
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim

spec:
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
	 storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
	  release: "stable"
	matchExpressions:
	  - {key: environment, operator: In, values: [dev]}
```
* accessMode, VolumeMode는 PV와 동일함
* resources는 PV와 같이, 필요한 볼륨의 사이즈를 정의
* selector를 통해서 볼륨을 선택할 수 있음
	- label selector 방식으로 이미 생성되어 있는 PV 중에, label이 매칭되는 볼륨을 찾아서 연결함 

#### 정적 설정
<!-- 외부 스토리지 서버 필요! -->

#### 동적 설정(Dynamic Provisioning)
* 정적 설정으로 PV를 수동으로 생성한후 PVC에 바인딩 한 후에, Pod에서 사용할 수 있지만, 
	- 쿠버네티스 1.6에서 부터 Dynamic Provisioning(동적 생성) 기능을 지원함
* 동적 생성 기능은 시스템 관리자가 별도로 디스크를 생성하고 PV를 생성할 필요 없이 PVC만 정의하면 이에 맞는 물리 디스크 생성 및 PV 생성을 자동화해주는 기능

![Dynamic Provisioning](./Kubernetes in Action_6장_볼륨/6장_01_DynamicProvisioning.png)
* PVC를 정의하면, PVC의 내용에 따라서 쿠버네티스 클러스터가 물리 Disk를 생성하고, 이에 연결된 PV를 생성함 
* 실제 환경에서는 성능에 따라 다양한 디스크(nVME, SSD, HDD, NFS 등)를 사용할 수 있음
	- 그래서 디스크를 생성할때, 필요한 디스크의 타입을 정의할 수 있는데,
	- 이를 storageClass라고 함
		+ PVC에서 storage class를 지정하면, 이에 맞는 디스크를 생성하게 됨 
	- Storage class를 지정하지 않으면, 디폴트로 설정된 storage class 값을 사용 

Ex) dynamic-pvc.yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mydisk
  
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
```
* 작성 후 실행
* 'StorageClass'가 없으면 'Pending'상태로 존재

Ex) dynamic-pvc를 사용할 포드 설정
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis

spec:
  containers:
    - name: redis
	  image: redis
      volumeMounts:
      - name: terrypath
		mountPath: /data/shared

  volumes:
    - name : terrypath
      persistentVolumeClaim:
        claimName: mydisk
```
* PV(StorageClass)가 없어 실행에 오류 발생

##### Storage Class 
* 정의한 스토리지 클래스는 PVC 정의시에, storageClassName에 적으면 PVC에 연결이 되고, 스토리지 클래스에 정해진 스펙에 따라서 물리 디스크와 PV를 생성하게 됨

Ex) AWS EBS 디스크에 대한 스토리지 클래스 지정한 예
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  zones: us-east-1d, us-east-1c
  iopsPerGB: "10"
```
* slow라는 이름으로 스토리지 클래스를 지정
* EBS 타입은 io1을 사용
	- GB당 IOPS는 10을 할당
	- 존은 us-east-1d와 us-east-1c에 디스크를 생성하도록 함

Ex) GCP의 Persistent Disk(pd)의 예
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  zones: us-central1-a, us-central1-b
```  
* slow라는 이름으로 스토리지 클래스를 지정
* pd-standard(HDD)타입으로 디스크를 생성
	- GB당 IOPS는 10을 할당
	- 존은 us-central1-a와 us-central1-b에 디스크를 생성하도록 함 
  
---
## 출처
[^조대협의 블로그]: https://bcho.tistory.com/1259 