Kubernetes 클러스터 구축하기: Quick Start
=========================================

#### 시나리오

1.	kubespray를 통해
2.	gcloud를 사용하여 3개의 gce instance를 생성하고
3.	kubernetes cluster를 구축하고
4.	node.js sample server를 docker image로 생성하고
5.	해당 이미지를 Google Container Registry(GCR)에 배포하고
6.	replicationController로 pod 생성하고
7.	nodePort로 service를 생성해서
8.	해당 웹페이지에 접속한다.

9.	환경:

	-	macos

### 준비

#### gcloud sdk 설치

[macos에서 gcloud 설치하기](https://cloud.google.com/sdk/docs/quickstart-macos)

### instance 생성

-	local에서 원격으로 gcp instance(gce)를 kube01,kube02,kube03 이름으로 생성
-	원하는 instance name 및 옵션 변경해 보자

```shell
gcloud compute instances create kube01  \   # kube02, kube03도 생성
--labels=username=jinwookchung  \
--zone=asia-northeast3-c  \
--machine-type=n1-highmem-4  \
--image-project=centos-cloud  \
--image-family=centos-7  \
--boot-disk-size=20GB  \
--scopes=cloud-platform  \
--tags=http-server,https-server
```

-	생성 확인

```shell
gcloud compute instances list
```

### ssh로 kube01에 접속

-	ssh key 생성

```shell
ssh-keygen
# 엔터,엔터 눌러 생성해도 되지만, 키 관리를 위해 키이름 변경 권고
```

-	gce metadata에 생성한 key 복사

```shell
cat ~/.ssh/id_rsa.pub
```

-	output을 복사하여, gce metadata ssh에 추가

<img src='./pictures/metadata-ssh.png'>

-	master로 사용할 kube01에 접속한다.

```shell
# private key와 key 생성시 사용한 username 사용
ssh -i ~/.ssh/id_rsa <username>@<kube01_external_ip>
```

### kube01의 key 복사

-	kubespray를 통한 쿠버 설치를 위해 kube01 키 사용
-	kube01에서 위의 과정 반복
-	metadata에 두개의 키가 추가 됐음을 확인

### kubernetes 설치

```shell
## kubespray 설치
sudo yum -y install python-pip git
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
sudo pip install -r requirements.txt

## sample을 복사하여 생성된 인스턴스 ip 변경
cp -r inventory/sample  inventory/my-cluster
cd inventory/my-cluster

## node 정보 변경
vi inventory.ini
# - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
[all]
 kube01 ansible_host=10.178.15.191   # hostname, 인스턴스의 internal ip 입력
 kube02 ansible_host=10.178.15.192
 kube03 ansible_host=10.178.15.193

[kube-master]
 kube01

[etcd]
 kube01

[kube-node]
 kube02
 kube03
...
# - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

## kubernetes 설치 (5~10분정도 소요)
ansible-playbook -i /home/jinwookchung/kubespray/inventory/my-cluster/inventory.ini -v --become --become-user=root /home/jinwookchung/kubespray/cluster.yml
```

### docker image 만들기

-	web 페이지 접속 시, hello world 문구와 hostname을 보여줄 server.js 생성

```js
var os = require('os');

var http = require('http');
var handleRequest = function(request, response) {
  response.writeHead(200);
  response.end("Hello World! I'm "+os.hostname());


  console.log("["+
                Date(Date.now()).toLocaleString()+
                "] "+os.hostname());
}
var www = http.createServer(handleRequest);
www.listen(8080);
```

-	Dockerfile 생성 (node carbon 사용)

```shell
FROM node:carbon
EXPOSE 8080
COPY server.js .
CMD node server.js > log.out
```

-	Docker 패키징 [(참고)](https://cloud.google.com/container-registry/docs/pushing-and-pulling)

```shell
docker build -t asia.gcr.io/<gcp_project_name>/hello-node:v1
# [asia 데이터 센터]/[해당프로젝트이름]/[이미지이름:버전]
```

-	로컬에서 Container Registry 인증

```shell
gcloud auth configure-docker
```

-	gcr에 이미지 저장

```shell
docker push asia.gcr.io/<gcp_project_name>/hello-node:v1
```

<img src='./pictures/gcr-image.png' width=400>

<br><br>

### 쿠버네티스에 서비스 배포하기

##### kube01에 kubectl alias 등록

-	kubespray로 설치하면 kubectl

```shell
sudo su
echo "alias kubectl='/usr/local/bin/kubectl'" >> ~/.bashrc
```

-	(option) 편의를 위해 kubectl 대신 k 사용

```shell
echo "alias k=kubectl" >> ~/.bashrc
echo "complete -F __start_kubectl k" >> ~/.bashrc
source ~/.bashrc
```

##### ReplicationController object를 등록하기 위해 hello-node-rc.yaml 생성

```shell
apiVersion: v1
kind: ReplicationController
metadata:
  name: hello-node-rc
spec:
  replicas: 3
  selector:
    app: hello-node
  template:
    metadata:
      name: hello-node-pod
      labels:
        app: hello-node
    spec:
      containers:
      - name: hello-node
        image: gcr.io/accu-platform/hello-node:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
```

##### service object를 등록하기 위해 hello-node-svc.yaml 생성

```shell
apiVersion: v1
kind: Service
metadata:
  name: hello-node-svc
spec:
  selector:
    app: hello-node
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8080
      nodePort: 32000
  type: NodePort
```
