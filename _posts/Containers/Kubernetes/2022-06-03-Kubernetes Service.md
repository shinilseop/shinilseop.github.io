---
title:  "Kubernetes Service (쿠버네티스 서비스)"
excerpt: "Kubernetes Serivce Notion, Example"

categories:
  - Kubernetes
tags:
  - [Container, Kubernetes, Object, Resource, Serivce]

toc: true
toc_sticky: true
 
date: 2022-06-03
last_modified_at: 2022-06-03

---

# 서비스(Service) : 포드를 연결하고 외부에 노출

- 사용자들이 포드들의 내부에 접근하기 위해 생성하는 오브젝트
- 쿠버네티스 리소스를 생성하는 yaml 파일에 containerPort 항목에 개방할 포트를 정하지만 바로 외부로 노출되는 것은 아님. 외부로 노출시키기 위해서는 서비스를 꼭 생성해야함

## 핵심 기능

- 여러 개의 포드에 쉽게 접근할 수 있도록 고유한 도메인 이름을 부여
- 여러 개의 포드에 접근할 때, 요청을 분산하는 로드 밸런서 기능을 수행
- 클라우드 플랫폼의 로드 밸런서, 클러스터 노드의 포트 등을 통해 포드를 외부로 노출

## 서비스의 타입

- ClusterIP
    - 쿠버네티스 네부에서만 포드들에 접근할 때 사용
    - 외부로 노출시키지 않기 때문에 내부에서만 작동되는 포드들에 적용
- NodePort
    - 포드에 접근할 수 있는 포트를 클러스터의 모든 노드에 동일하게 개방
    - 외부에서 접근이 가능해지는 서비스 타입
    - 접근 포트는 랜덤으로 정해지지만 특정 포트로 접근하도록 설정 가능
- LoadBalancer
    - 클라우드 플랫폼에서 제공하는 로드밸런서를 동적으로 프로비저닝해서 포드에 연결
    - NodePort와 마찬가지로 외부에서 접근이 가능해지는 서비스 타입
    - 일반적으로 AWS, GCP 등과 같은 클라우드 플랫폼 환경에서만 사용 가능
    

## ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hostname-svc-clusterip
spec:
  ports:
    - name: web-port
      port: 8080
      targetPort: 80
  selector:
    app: webserver
  type: ClusterIP
```

- spec.selector
    - 서비스에서 어떤 라벨을 가지는 포드에 접근할 수 있게 만들 것인지 결정
- sepc.ports.port
    - 쿠버네티스 내부에서만 사용할 수 있는 고유한 IP(ClusterIP)를 할당받음
    - port 항목에는 서비스의 IP에 접근할 때 사용할 포트를 설정
- spec.ports.targetPort
    - selector 항목에서 정의한 라벨에 의해 접근 대상이 된 파드들이 내부적으로 사용하고 있는 포트 명시
- spec.type
    - 서비스의 타입을 명시
    
- 생성 및 확인

![clusterip-svc-create](../../../assets/images/container/kubenetes/service/Untitled%205.png)

- curl이 존재하는 우분투 이미지 생성 후 좀 전에 생성한 ClusterIP의 개방포트(8080)로 요청

![ubuntu:curl-create](../../../assets/images/container/kubenetes/service/Untitled%206.png)

![send-get](../../../assets/images/container/kubenetes/service/Untitled%207.png)

![curl-loadbalancer](../../../assets/images/container/kubenetes/service/Untitled%208.png)

- 여러번 요청을 보내보면 별도의 설정을 하지 않았음에도 서비스는 연결된 포드에 대해 로드 밸런싱을 수행하는 것을 볼 수 있음.

![curl-dns](../../../assets/images/container/kubenetes/service/Untitled%209.png)

- IP 뿐만 아니라 서비스 이름 그 자체로도 접근이 가능
- 쿠버네티스가 내부 DNS를 구동하고 있기 때문에 가능
- 실제로 여러 포드가 클러스터 내부에서 서로를 찾아 연결할 때에는 서비스의 이름과 같은 도메인 이름을 사용하는 것이 일반적

- 서비스의 라벨 셀렉터와 포드의 라벨이 매칭돼 연결되면 쿠버네티스는 자동으로 엔드포인트라고 부르는 오브젝트를 별도로 생성

![endpoint](../../../assets/images/container/kubenetes/service/Untitled%2010.png)

- 엔드포인트 또한 오브젝트이기에 이론상으로 서비스와 엔드포인트를 따로 만드는 것이 가능

- 삭제 : `kubectl delete svc hostname-svc-clusterip`

## NodePort

```yaml
# hostname-svc-nodeport.yaml

apiVersion: v1
kind: Service
metadata:
  name: hostname-svc-nodeport
spec:
  ports:
    - name: web-port
      port: 8080
      targetPort: 80
  selector:
    app: webserver
  type: NodePort
```

- ClusterIP 만들때와 같고 마지막에 type만 다름

- 생성 후 개방된 포트로 접근해보기
    - `kubectl apply -f hostname-svc-nodeport.yaml`

![nodeport-create](../../../assets/images/container/kubenetes/service/Untitled%2011.png)

- 개방된 32565 포트는 모든 노드에서 접근할 수 있는 포트를 의미
- 단, GKE에서 쿠버네티스를 사용하고 있는 경우, 각 노드에 랜덤한 포트에 접근하기 위해 별도로 방화벽을 설정해야함
- AWS에서도 마찬가지로 Security Group에 별도의 Inbound 규칙을 추가하지 않으면 NodePort 통신이 실패할 수도 있음

```bash
$gcloud compute firewall-rules create temp-nodeport-svc --allow=tcp:32565 # 추가
$gcloud compute firewall-rules delete temp-nodeport-svc # 삭제
```

- 각 노드에서 개방되는 포트는 기본적으로  30000~32768 포트 중에 랜덤으로 선택되지만, YAML 파일에 nodePort 항목을 정의하면 원하는 포트 선택 가능

```yaml
...
spec:
  ports:
    - name: web-port
      port: 8080
      targetPort: 80
			nodePort: 31000
...
```

![nodeport-clusterip](../../../assets/images/container/kubenetes/service/Untitled%2012.png)

- NodePort 타입의 서비스를 만든 것을 보면 하나 이상한 점이 있음, 바로 ClusterIP가 할당 된것
- 실제로 ClusterIP에 접근했던 것처럼 curl을 이용해 접근해보면 접근 가능
- 사실 NodePort가 ClusterIP 기능을 포함하고 있기 때문
- NodePort 타입의 서비스를 생성하면 자동으로 ClusterIP 기능을 사용할 수 있기 때문에 쿠버네티스 클러스터에서 서비스의 내부 IP와 DNS 이름을 사용해 접근이 가능한 것
- 그러므로 NodePort 타입의 서비스는 내부, 외부 모두 접근이 가능

- 기본적으로 NodePort가 사용 가능한 포트 범위는 30000~32768 이지만, API 서버 컴포넌트의 실행 옵션을 변경하면 원하는 포트 범위로 설정 가능
    - `—service-node-port-range=30000-35000`
- 특정 클라이언트가 같은 포드로부터만 처리되게 하려면 서비스의 YAML 파일에서
- sessionAffinity 항목을 ClientIP로 설정

```yaml
...
spec:
	sessionAffinity: ClientIP
...
```

- 실제 운영 환경에서 NodePort 서비스를 외부에 제공하는 경우는 많지 않음
- 포트 번호를 80, 443으로 설정하기에 적합하지 않은 경우도 많고 SSL 인증서 적용, 라우팅 등과 같은 복잡한 설정을 서비스에 적용하기가 어렵기 때문
- Ingress 라고 부르는 쿠버네티스 오브젝트에서 간접적으로 사용되는 경우가 많음
- 지금은 “외부 요청을 실제로 받아들이는 관문” 정도로 알면 됨
- LoadBalancer와 NodePort를 합치면 Ingress 오브젝트를 사용 가능

## LoadBalancer

- 서비스의 생성과 동시에 로드 밸런서를 새롭게 생성해 포드와 연결
- NodePort를 사용할 때는 각 노드의 IP를 알아야만 포드에 접근할 수 있었찌만, 로드밸런서의 경우 클라우드 플랫폼으로 부터 도메인 이름과 IP를 할당받기 때문에 더욱 쉽게 포드에 접근 가능
- 단, 로드밸런서의 경우 로드 밸런서를 동적으로 생성하는 기능을 제공하는 환경에서만 사용할 수 있음
- 일반적으로 AWS, GCP 등과 같은 클라우드 플랫폼 환경에서만 로드밸런서를 사용 가능하며 가상머신이나 온프레미스 환경에서는 사용하기 어려울 수 있음 (MetalLB, 오픈스택의 LBaaS 등과 같이 온프레미스에서도 가능하게 하는 방법들이 존재하긴 함)

```yaml
# hostname-svc-lb.yaml

apiVersion: v1
kind: Service
metadata:
  name: hostname-svc-lb
spec:
  ports:
    - name: web-port
      port: 80
      targetPort: 80
  selector:
    app: webserver
  type: LoadBalancer
```

- `kubectl apply -f hostname-svc-lb.yaml`

![loadbalancer-create](../../../assets/images/container/kubenetes/service/Untitled%2013.png)

![get-objects](../../../assets/images/container/kubenetes/service/Untitled%2014.png)

- 현재 본인은 로컬에 Docker for Windows의 Kubetnetes를 사용중이어서 EXTERNAL-IP가 localhost로 표시됨.
- AWS를 사용할 경우 a34kjsf….elb.amazonaws.com 라는 도메인으로 뜰것임
    
    ![connect-localhost](../../../assets/images/container/kubenetes/service/Untitled%2015.png)
    
- 뜨기는 뜨지만 로드밸런싱은 안되고 계속 저 pod만 연결됨

- 아님; 재차 확인 결과 pending이 뜨는것이 정상 어케한거지 ;

![externalip-pending](../../../assets/images/container/kubenetes/service/Untitled%2016.png)

- localhost 또한 접근 안됨.

- EXTERNAL-IP 외에 PORT를 확인해보면 80 : **32718**가  존재하는데 이는 마치 NodePort의 타입과 비슷해보임
- 포트를 개방한 이유는 로드밸런서 서비스가 포드로 요청을 전송하는 원리에 있음
    1. LoadBalancer 타입의 서비스가 생성됨과 동시에 모든 워커 노드는 포드에 접근할 수 있느 랜덤한 포트를 개방 (ex> 32718)
    2. 클라우드 플랫폼에서 생성된 로드 밸런서로 요청이 들어오면 이 요청은 쿠버네티스의 워커 노드 중 하나로 전달되며, 이때 사용되는 포트는 1번에서 개방한 포트를 사용
    3. 워커 노드로 전달된 요청은 포드 중 하나로 전달되어 처리됨
- 로드밸런서 타입으로 생성했지만, NodePort의 간접적인 기능 또한 자동으로 사용할 수 있는 셈
- 또한, 클라우드 로드밸런서의 유형을 바꿔서 생성할 수도 있음

```yaml
...
apiVersion: v1
kind: Service
metadata:
  name: hostname-svc-lb
	**annotations:
		service.beta.kubernetes.io/aws-load-balancer-type: "nlb"**
spec:
  ports:
    - name: web-port
      port: 80
      targetPort: 80
  selector:
...
```

- annotation은 라벨처럼 해당 리소스의 추가적인 정보를 나타내기 위한 키:값 쌍으로 이루어짐
- 특정 용도로 사용되도록 쿠버네티스에 미리 정의된 몇 가지 주석이 존재하며 리소스에 특별한 설정값을 부여하기 위해 사용됨

- 삭제
    - `kubectl delete -f hostname-svc-lb.yaml`
    

## 트래픽의 분배를 결정하기

### externalTrafficPolicy : 트래픽 처리를 Local인지 Cluster인지 결정

- 로드밸런서 타입의 서비스를 사용하게 되면 외부에서 들어온 요청은 각 노드 중 하나로 보내지며, 그 노드에서 다시 포드로 접근하게 됨
- 하지만, 이러한 요청이 꼭 효율적이다 라는 보장은 없음

![externalTrafficPolicy-1](../../../assets/images/container/kubenetes/service/Untitled%2017.png)

A로 요청이 들어왔지만 로드밸런서에 의해 B로 전송되는 모습

- B로 전달하게 되면 불필요한 네트워크 홉(hob)이 한 단계 발생되고 노드 간의 리다이렉트가 발생하게 되어 트래픽의 출발지 주소가 바뀌는 SNAT이 발생하고, 이로 인해 클라이언트의 IP 주소 또한 보존되지 않음
- A로 들어온 요청을 굳이 B로 전달하지 않고 A 노드에서 처리할 수는 없을까 ?
- externalTrafficPolicy를 설정해서 바꿀 수있음.
- kubectl get -o yaml 명령어를 통해 서비스의 모든 속성을 출력해 볼 수 있음.
    - `kubectl get svc hostnamesvc-nodeport -o yaml`

```yaml
# kubectl get svc hostnamesvc-nodeport -o yaml

apiVersion: v1
kind: Service
metadata:
	annotaions:
...
	**externalTrafficPolicy: Cluster**
...
```

- 이를 클러스터가 아니라 Local로 설정하게 되면 포드가 생성된 노드에서만 포드로 접근이 가능해지며 로컬 노드에 위치한 포드 중 하나로 요청이 전달됨
- 추가적인 네트워크 홉이 발생하지 않으며, 전달되는 요청의 클라이언트 IP 또한 보존이 됨

![externalTrafficPolicy-2](../../../assets/images/container/kubenetes/service/Untitled%2018.png)

### ExternalName : 요청을 외부로 리다이렉트를하는 서비스

- 쿠버네티스를 외부 시스템과 연동해야 할 때가 생길 수도 있음
- 서비스가 외부 도메인을 가르키도록 설정하는 방법

```yaml
apiVersion: v1
kind: Service
metadata:
	name: externalname-svc
spec:
	type: ExternalName
	externalName: my.database.com
```

- 리소스 정리
    - kubectl delete -f ./

## 온프레미스 환경에서  LoadBalancer 타입의 서비스 사용하기

### MetalLB 설치

[MetalLB, bare metal load-balancer for Kubernetes](https://metallb.universe.tf/installation/)

- kube-system의 kube-proxy 설정 변경하기
    - `kubectl edit configmap -n kube-system kube-proxy`
    - 변경하는 이유 : 쿠버네티스 1.14.2 버전부터 kube-proxy IPVS mode를 사용하려면 strictARP모드를 켜줘야 함. 기본적으로 strictARP를 활성화 하기 때문에 kube-router를 서비스 프록시로 사용하는 경우에는 필요하지 않음.

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```

![수정전](../../../assets/images/container/kubenetes/service/Untitled%2019.png)

수정전

![수정후](../../../assets/images/container/kubenetes/service/Untitled%2020.png)

수정후

- 구성을 kubeadm-config에 추가하는 법

```bash
# **see what changes** would be made, returns nonzero returncode if different
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

# actually **apply** the changes, returns nonzero returncode on errors only
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

- 매니페스트 적용을 통한 MetalLB 설치

```bash
# namespace 만들기
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml

# 구성요소 설치
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
```

![metallb-install](../../../assets/images/container/kubenetes/service/Untitled%2021.png)

- metallb-system 네임스페이스 아래의 클러스터에 MetalLB가 배포됨
- 매니페스트 구성요소
    - metallb-system/controller 배포 : IP 주소 할당을 처리하는 클러스터 전체 컨트롤러
    - metallb-system/speaker 데몬셋 : 서비스에 도달할 수 있도록 내가 선택한 프로토콜을 알려주는 컴포넌트
    - 서비스 계정 : 구성 요소가 작동하는데 필요한 RBAC 권한과 컨트롤러와 스피커에 대한 계정
- 설치 매니페스트에는 구성파일이 포함되어 있지 않음

```bash
# Secret 생성 
# error: failed to create secret secrets "memberlist" already exists
# kubectl get secret -n metallb-system (확인결과 기본적으로 생성되는거 같음)
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

```yaml
# 라우팅 처리를 위한 configmap 생성

apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 10.1.0.2-10.1.0.100
```

- 생성 후 아까 생성한 LB를 확인해보면 EXTERNAL-IP가 연결됨

![metallb-configmap-create](../../../assets/images/container/kubenetes/service/Untitled%2022.png)

![access-metallb](../../../assets/images/container/kubenetes/service/Untitled%2023.png)

- IPv4 주소나 localhost로 접근해보면 로드 밸런싱 해서 보여주는걸 확인 가능