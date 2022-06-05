---
title:  "Kubernetes Config, Secret (쿠버네티스 컨피그, 시크릿)"
excerpt: "Kubernetes Config, Secret Notion, Example"

categories:
  - Kubernetes
tags:
  - [Container, Kubernetes, Object, Resource, Config, Secret]

toc: true
toc_sticky: true
 
date: 2022-06-06
last_modified_at: 2022-06-06

---

# Configmap, Secret : 설정값을 포드에 전달

- 개발한 애플리케이션의 대부분은 설정값을 가지고 있음
- 로깅 레벨을 정하는 LOG_LEVEL=INFO나 MYSQL_PASSWORD=1234 과 같이 키-값 형태의 설정을 사용할 수도 있고, Nginx 웹 서버가 사용하는 nginx.conf처럼 완전한 하나의 파일을 사용해야 할 수도 있음
- 도커의 이미지에 설정 값 또는 설정 파일을 전달하는 것이 가장 확실하지만 이미지는 정적이므로 유동적이게 변화할수가 없다는 단점이 있음

- 이에 대한 대안으로 포드를 정의하는 YAML 파일에 환경 변수를 직접 적어 놓는 하드 코딩 방식을 사용 가능

```jsx
...
	spec:
		containers:
		- name: nginx:
			env:
			- name: LOG_LEVEL
				value: INFO
			image: nginx:1.10
...
```

- 하지만, 상황에 따라서는 동일한 YAML 파일에서 환경 변수의 값만 다른  경우가 존재할 수 있음 (ex> 운영 환경과 개발 환경에서 각각 디플로이먼트를 생성할 때)
- 이러한 이유에 의해 쿠버네티스에서는 YAML 파일과 설정 값을 따로 분리할 수 있는 컨피그맵(Configmap)과 시크릿(Secret)이라는 오브젝트를 제공
- 컨피그맵에는 설정값을, 시크릿에는 노출되어서는 안되는 비밀값을 저장

## Configmap

- 일반적인 설정값을 담아 저장할 수 있는 쿠버네티스 오브젝트이며, 네임스페이스에 속하기 때문에 네임스페이스별로 존재
- 생성
    - `kubectl create configmap {컨피그맵-이름} {각종-설정값들}`
    - `kubectl create configmap log-level-configmap —from-literal LOG_LEVEL=DEBUG`
    - `kubectl create configmap start-k8s —from-literal k8s=kubernetes —from-literal container=docker`

![kubectl-create-configmap](../../../assets/images/container/kubenetes/config_secret/Untitled%204.png)

- YAML로 확인

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%205.png)

- 컨피그맵 값을 컨테이너의 환경 변수로 사용
    - 키-값이 환경변수로 저장되기 때문에 echo $LOG_LEVEL 과 같은 방식으로 확인 가능
- 컨피그맵 값을 포드 내부의 파일로 마운트해 사용
    - 컨피그맵을 /etc/config/log_level 이라는 파일로 마운트하면 저장이 되고 사용 가능
    - 만약, nginx.conf 등의 파일을 통해 설정값을 읽어 들인다면 이 방법을 사용하는 것이 좋음
- 컨피그맵의 데이터를 컨테이너의 환경 변수로 가져오기

```bash
apiVersion: v1
kind: Pod
metadata:
  name: container-env-example
spec:
  containers:
    - name: my-container
      image: busybox
      args: ['tail', '-f', '/dev/null']
      envFrom:
        - configMapRef:
            name: log-level-configmap
        - configMapRef:
            name: start-k8s
```

- 생성 후 확인

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%206.png)

- 컨피그맵에서 선택적으로 변수를 가져오기

```bash
apiVersion: v1
kind: Pod
metadata:
  name: container-env-example2
spec:
  containers:
  - name: my-container2
    image: busybox
    args: ['tail', '-f', '/dev/null']
    env:
    - name: ENV_KEYNAME_1
      valueFrom:
        configMapKeyRef:
          name: log-level-configmap
          key: LOG_LEVEL
    - name: ENV_KEYNAME_2
      valueFrom:
        configMapKeyRef:
          name: start-k8s
          key: k8s
```

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%207.png)

- 컨피그맵의 내용을 파일로 포드 내부에 마운트하기

```bash
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod
spec:
  containers:
  - name: my-container
    image: busybox
    args: ["tail", "-f", "/dev/null"]
    volumeMounts:
    - name: configmap-volume
      mountPath: /etc/config
  
  volumes:
    - name: configmap-volume
      configMap:
        name: start-k8s
```

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%208.png)

- 이와 같이 쿠버네티스 리소스 데이터를 내부 디렉터리에 위치시키는 것을 공식 문서에서는 투사(Projection)한다고 표현함

- 선택적으로 볼륨에 마운트 시키기

```bash
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod
spec:
  containers:
  - name: my-container
    image: busybox
    args: ["tail", "-f", "/dev/null"]
    volumeMounts:
    - name: configmap-volume
      mountPath: /etc/config
  
  volumes:
    - name: configmap-volume
      configMap:
        name: start-k8s
        items: # 가져올 키 값
        - key: k8s # 키 이름
          path: k8s_fullname # 최종적으로 디렉토리에 위치할 파일의 이름
```

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%209.png)

- 최종 파일 이름이 k8s_fullname 으로 명시됨

### 파일로부터 컨피그맵 생성하기

- —from-literal 이 아닌 —from-file 옵션을 사용하면 가능
- 여러 번 사용해서 여러 개의 파일을 컨피그맵에 저장하는 것도 가능
- `kubectl create configmap {컨피그맵-이름} —from-file {파일 이름} …ㄷ`

```bash
# index.html

Hello, world
```

- `kubectl create configmap index-file —from-file index.html`
- `kubectl create configmap index-file —from-file myindex=index.html` → 이름 명시

- env 파일을 활용해서 만들기

```bash
# multiple-keyvalue.env

mykey1=myvalue1
mykey2=myvalue2
mykey3=myvalue3
```

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%2010.png)

- index.html 이나 nginx.conf와 같이 정적 파일을 포드에 제공하려면 —from-file 옵션을 쓰는 것이 편리할 수도 있고, 여러 개의 환경 변수를 포드로 가져와야 한다면 —from-env-file 옵션을 쓰는 것이 편리할 수도 있음

### YAML 파일로 컨피그맵 정의하기

- `kubectl create cm my-configmap --from-literal mykey=myvalue --dry-run -o yaml`
    - —dry-run -o yaml 옵션을 통해서 컨피그맵을 생성하지 않고 YAML파일로 확인 가능
    - dry run 이란 특정 작업의 실행 가능 여부를 검토하는 명령어 또는 API를 의미
    - 실제로 생성하지는 않고 실행 가능 여부 등을 파악

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%2011.png)

## Secret

- 시크릿은 SSH키, 비밀번호 등과 같이 민감한 정보를 저장하기 위한 용도로 사용되며, 네임스페이스에 종속되는 쿠버네티스 오브젝트
- 시크릿과 컨피그맵은 사용 방법이 매우 비슷, 컨피그맵에 설정값을 저장했던 것처럼 시크릿 또한 문자열 값 등을 똑같이 저장 가능 + 포드에 제공
- 그렇지만 민감한 정보를 저장하기 위해 컨피그맵보다 좀 더 세분화된 사용 방법을 제공

- 생성
- `kubectl create secret generic my-password --from-literal password=1q2w3e4r`
    - 기본적인 시크릿 파일 생성방법
- 여러 개의 환경을 설정하기

```bash
echo mypassword > pw1 %% echo yourpassword > pw2
kubectl create secret generic our-password --from-file pw1 --from-file pw2
```

- 생성 확인

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%2012.png)

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%2013.png)

- 실제 생성된 것을 확인해보면 비밀번호가 base64 기반으로 인코딩 되어서 저장됨
- 이렇게 생성한 시크릿은 컨피그맵과 비슷하게 사용 키-값 데이터를 포드에 환경 변수로 설정도 가능하고, 특정 경로의 파일로 포드 내에 마운트도 가능
- 단, 사용되는 YAML 항목이 configmap이 아니라 secret 임

- 생성된 시크릿을 포드에 적용시키기

```bash
# env-from-secret.yaml

apiVersion: v1
kind: pod
metadata:
  name: secret-env-example
spec:
  containers:
  - name: my-container
    image: busybox
    args: ['tail', '-f' '/dev/null']
    envFrom:
    - secretRef:
        name: my-password
```

- 선택적으로 가져와서 적용시키기

```bash
# selective-env-from-secret.yaml

apiVersion: v1
kind: Pod
metadata:
  name: secret-env-example
spec:
  containers:
  - name: my-container
    image: busybox
    args: ['tail', '-f', '/dev/null']
    env:
    - name: YOUR_PASSWORD
      valueFrom:
        secretKeyRef:
          name: our-password
          key: pw2
```

- 시크릿 파일을 볼륨으로 마운트 시키기

```bash
# volume-mount-secret.yaml

apiVersion: v1
kind: Pod
metadata:
  name: secret-env-example
spec:
  containers:
  - name: my-container
    image: busybox
    args: ['tail', '-f', '/dev/null']
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret
      
  volumes:
  - name: secret-volume
    secret:
      secretName: our-password
```

- 선택적으로 가져와서 마운트 시키기

```bash
# selective-mount-secret.yaml

apiVersion: v1
kind: Pod
metadata:
  name: secret-env-example
spec:
  containers:
  - name: my-container
    image: busybox
    args: ['tail', '-f', '/dev/null']
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret
      
  volumes:
  - name: secret-volume
    secret:
      secretName: our-password
      items:
        - key: pw1
          path: password1
```

- 생성할때 base64 기반으로 인코딩된 값을 입력했더라도 시크릿을 포드의 환경 변수나 볼륨 파일로서 가져오면 base64로 디코딩된 원래의 값을 사용

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%2014.png)

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%2015.png)

- 생성한 시크릿의 타입을 확인해보면 Opaque로 설정이 되어있음
- 별도로 시크릿의 종류를 명시하지 않으면 자동으로 설정되는 타입으로, 사용자가 정의하는 데이터를 저장할 수 있는 일반적인 목적의 시크릿
- kubectl create 명령어로 시크릿을 생성할 때 generic이라고 명시했던 것이 바로 Opaque 타입에 해당
- 시크릿을 generic으로 생성할 때는 컨피그맵과 큰 차이가 느껴지지 않을 수도 있음
- 그러나 시크릿은 사용 용도에 따라 여러 종류를 설정할 수 있으며, 그 중 하나가 비공개 레지스트리(Private Registry)에 접근할 때 사용하는 인증 설정 시크릿

- 단일 도커 서버에서는 `docker login` 를 이용해서 인증을 진행
- 쿠버네티스에서는 로그인 대신 레지스트리 인증 정보를 저장하는 별도의 시크릿을 생성해 사용
    1. docker login 명령어로 로그인에 성공했을 때 도커 엔진이 자동으로 생성하는 ~/.docker/config.json 파일을 사용하는 것
        
        ```bash
        kubectl create secret generic registry-auth \
        --from-file=.dockerconfigjson=/root/.docker/config.json \
        --type=kubernetes.io/dockerconfigjson
        ```
        
    2. 시크릿을 생성하는 명령어에서 직접 로그인 인증 정보를 명시할 수도 있음. 각 옵션에 적절한 인자를 입력하면 되며, —docker-username과 —docker-password 옵션은 필수
        
        ```bash
        kubectl create secret docker-registry registry-auth-by-cmd \
        --docker-username=alicek106 \
        --docker-password=1q2w3e4r
        ```
        
        - —docker-server는 필요에 따라 사용하는데 기본적으로는 도커 허브(docker.io)를 사용하도록 설정되어 있기때문에 다른 사설 레지스트리를 사용하려면 변경해야함
        
- YAML 파일에서 생성한 시크릿을 통해 인증해서 이미지를 Pull 하는 법

```bash
apiVersion: apps/v1
kind: Deployment
...
		sepc:
			containers:
			- name: test-container
				image: exampleImage
			imagePullSecrests:
			- name: {secret-name}
```

```bash
🗒️
기본적으로는 워커 서버에 이미지가 존재하지 않을 때만 이미지를 받아오도록 설정돼 있지만, 
imagePullPolicy 항목을 수정하여 이미지를 받아오는 설정을 변경 가능
```

### TLS 타입의 시크릿 사용하기

- 시크릿은 TLS 연결에 사용되는 공개키, 비밀키 등을 쿠버네티스에 자체적으로 저장할 수 있도록 tls 타입을 지원
- 포드 내부의 애플리케이션이 보안 연결을 위해 인증서나 비밀키 등을 가져와야 할 때 시크릿의 값을 포드에 제공하는 방식으로 사용 가능
- 추후 고급 기능을 사용하게 되면 tls 타입의 시크릿을 사용하는 것이 더욱 편리해질 것임

**tls 타입의 시크릿 사용하기**

1. 보안 연결에 사용할 키 페어 준비
    
    ```bash
    $ openssl req -new -newkey rsa:4096 -days 365 -nodes \
    -x509 -subj "/CN=example.com" -keyout cert.key -out cert.crt # 키 페어 생성
    
    $ ls
    cert.crt cert.key ..
    
    $ kubectl create secret tls my-tls-secret \
    --cert cert.crt --key cert.key
    ```
    

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%2016.png)

- `kubectl get secrets my-tls-secret -o yaml` 을 통해서 생성된 키 확인 가능
- base64로 인코딩 되어서 저장됨

### 조금 더 쉽게 컨피그맵과 시크릿 리소스 배포하기

- 앞서 시크릿을 생성하기 위해 CLI를 통해서 생성했지만, 이를 YAML 파일로 배포하려면 시크릿의 데이터를 YAML 파일에 함께 저장해 둬야 함
- dry run 명령어를 통해 출력되는 YAML 파일 내용을 저장해 사용할 수도 있음

```bash
apiVersion: v1
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUVxRENDQXBBQ0NRRHUwNTVhN3JuRV
NEQU5CZ2txaGtpRzl3MEJBUXNGQURBV01SUXdFZ1lEVlFRRERBdGwKZUdGdGNHeGxMbU52YlRBZUZ3MHl
NakEyTURReE5UVTRNemxhRncweU16QTJNRFF4TlRVNE16bGFNQll4RkRBUwpCZ05WQkFNTUMyVjRZVzF3
YkdVdVkyOXRNSUlDSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQWc4QU1JSUNDZ0tDCkFnRUF2c3lqcWlVY
2U1ZjBJOFlaNWVVdDhhcHhwTWdJSEJ6VzFCdUFpSWpNM2lXN016SlZUNGRRd25QZ2JSdmoKVjZoL3g4eW
haTUtyWjlJNzIwTkhadHhIUGE2ajRpVkZvaUd1OVNvYlZWaGxvVVZNVkpFbEZBMlkxYmpKbHhUeQpPN1V
iUDJiclBXMS9DQjNWNzJPckZCMTJ6Q2dVRUlqNXNWcUpuSldBL2c3emd0MXVBVDhHdnp3NkpOVzQ5YkVK
CnIxNFhrYzdwc1FMckdiNVFiZGNGWHgwTUZqL2pZb3Zaci9ja09raHRUS3N4VTNKR0hRd0U5K05OcXJFN
lJpS20KMTU3WGwvbExycThxdnQxbmQzNTBqYU9xWHJkcitadHcvQmJlZWgxQ1ZxeFFYaU4xdlowMGE4ZF
Qxd1loTnIwYwpDWmhxUGh1QVdsYXJzeng1eEEyZWxmSEJrNS9ZaVJmaFM4Z0h5TDlqbzZaNVpBaUpFeFR
MSEZycVhuTERzYUk2Ck5WOUF6aEpKTEJPZnZtUHVNTFFFM0xWQUdQbUdHTUtOYmlJUU1rMnhJN3F4VUdX
UDFQNi9EV3RidlZtZkFJVVQKcmlNTGprV3p4eUxOZEhUTGtGcWY3aXpZd2hLYTUwRjk4aVhYbGdIV21xd
TlleGJ3enlsbUFRK2thMkpJRFdVdAorS1pma2h6NDh6L2F2MjM4ODhUMFlNQ2MwU2VBQ0w4QmN0VkJ5OE
lJcEY4UEZ5blhjMFpBNmRUcHdjTDVqcUNICnZONk9NY0ZNWFpUOHhjZHg2b0k4Y2lXTW5WVTZMUFlQSHp
JV0ZuTlRWNTg3ZHBpQTRVUUpGZGEzbHdPTUpqanAKWmdwdUtrMS92RE5jOE1VZWc0UnNBL29mYmFITTFy
bFQ3UFQwMmN6UGRuM0FCVUVDQXdFQUFUQU5CZ2txaGtpRwo5dzBCQVFzRkFBT0NBZ0VBSEp5SGcxZy9uU
1Y3VHBOcUVsZUZsTldlOEk4dHRWSHQvc1ZiRHhxNXdPTVFKZC81CjRyakJzMkpQZ0RhVEk4YjZyWDFpcH
VEME5hREFHdTRwMzJIKzZ2cFZqT3RIclJsd3lKVzdRZ3ZXUHVleXpiU2EKcTc0NTJaV3FHeUhibjgxSEM
zQ2l5SnpJZTBWTVJacEZUY3NnUDF5VmpmNTZlWVpEWEd6OU52NUwwcUJqbmNKSgp0WGQvUEgrbDd5L25j
ZmdWTldjenljdFNMaXB1Y2FIR3NkZ3R2N3VuUFJmSEZxNDVCZUtJOGZ4VDgxb2hoNU9hCnRaby8yNEZqY
VZmajdtczRJWkJPWC9KRWEvVUZNZnZldXJwb0lRSjd3YnQxU2pHWlUzRHUvK2NyR2x1YzNTQm8KVEdrY0
9XMllRNC9ERTVXZ0pBSjNwOHlhcnBOc0NTTTJsV3RLWGxnbGxxVW9BdVduWFlsWkswTHRPUW91NXd3VQp
lbDlIelBVa09RMzY3eEUyaHRPOVkxSll1dG9ZVE4zUEc3bTExZHNPNzBpRTlMaDkxb1RaS3RPY2cwemNaS
jRqCmtuNmloMGtUalhnSWUzVTE4RHJiVVE1UTJxZEpLVnp0OFVqU0FYSnRBTzZ6TG1tVzF3dGlmYjBaTD
BRa2haaHUKZ3hMcjFpL3Y2b0l5dFcrUGd0c1B1TkFHUk9VeHY4SW5QbFpUZ2c1VDhqeXNucXdCMmk5Z3F
XclNNTmxNKzlObgp2QWtNd2V6SnkxSGRRajJ3Y1BVS2dYMTVmTzRSMXA2Nkp6cVVBSUVBdFNmNTJoaFFn
SWNtdmRtVnBsNHArbWlsCkJaUWJLWWVkTFJ4SFRhYmxMWUhkSkNpdyt2ZU52MFdmNXFzK3BZd0RXS1N3Q
k1LZ2pLRmxtRzk5dUs4PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUpRd0lCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQ1Mwd2dna3BBZ0VBQW9JQ0FRQyt6S09xSlJ4N2wvUWoKeGhubDVTM3hxbkdreUFnY0hOYlVHNENJaU16ZUpic3pNbFZQaDFEQ2MrQnRHK05YcUgvSHpLRmt3cXRuMGp2YgpRMGRtM0VjOXJxUGlKVVdpSWE3MUtodFZXR1doUlV4VWtTVVVEWmpWdU1tWEZQSTd0UnMvWnVzOWJYOElIZFh2Clk2c1VIWGJNS0JRUWlQbXhXb21jbFlEK0R2T0MzVzRCUHdhL1BEb2sxYmoxc1FtdlhoZVJ6dW14QXVzWnZsQnQKMXdWZkhRd1dQK05paTltdjl5UTZTRzFNcXpGVGNrWWREQVQzNDAycXNUcEdJcWJYbnRlWCtVdXVyeXErM1dkMwpmblNObzZwZXQydjVtM0Q4RnQ1NkhVSldyRkJlSTNXOW5UUnJ4MVBYQmlFMnZSd0ptR28rRzRCYVZxdXpQSG5FCkRaNlY4Y0dUbjlpSkYrRkx5QWZJdjJPanBubGtDSWtURk1zY1d1cGVjc094b2pvMVgwRE9Fa2tzRTUrK1krNHcKdEFUY3RVQVkrWVlZd28xdUloQXlUYkVqdXJGUVpZL1UvcjhOYTF1OVdaOEFoUk91SXd1T1JiUEhJczEwZE11UQpXcC91TE5qQ0Vwcm5RWDN5SmRlV0FkYWFxNzE3RnZEUEtXWUJENlJyWWtnTlpTMzRwbCtTSFBqelA5cS9iZnp6CnhQUmd3SnpSSjRBSXZ3RnkxVUhMd2dpa1h3OFhLZGR6UmtEcDFPbkJ3dm1Pb0llODNvNHh3VXhkbFB6RngzSHEKZ2p4eUpZeWRWVG9zOWc4Zk1oWVdjMU5Ybnp0Mm1JRGhSQWtWMXJlWEE0d21PT2xtQ200cVRYKzhNMXp3eFI2RApoR3dEK2g5dG9jeld1VlBzOVBUWnpNOTJmY0FGUVFJREFRQUJBb0lDQUUwWGdrbU5GU1ViRUpvandQTVMxcTErCm9NeGp4bU1WZy9mUDVPOUYxd0VyWGFnaC9qWlVCbDJMVkhMQmdlbzVPdWdQMW1aUUFkSEJNRTQzc1BIdXJ4cE4KSmdxSjVNak5zMU43Mys5citDUmhTNllmdjB1Szh1WG45QXdIZXBpRlpLMEplS01wU3RxTXM1UTJRVG12YmdDdgpjT3Y5YkdZc25zMlYycmpNY2JlK29HUUFnMGxobkZ5bHZrWUhjbEpaUWt5M3ZkUzN5U1p0cnpHeVg0ayt3MU42CkhQWUVhOENkcXhXaGpnZ2NZNkhEMm5DQ0dyL09KK09BR0h1ZUpLdWFrcUhsS0o4OFI1azIvRWRiNE53WjlReXcKTUFCNnZmd2RnV2IxeTRnWnQ1OVIxSkQ0bytXb2RFZTlRazVMdXJobHpRZXJOUUZMWTdUWUNwc2NwYWNRTjVZcQpjaFd0SU1xdjJNMXl1NEhnRm1MNXhHeVhqOVhIeGJoR1htTG9xY25XaVhzYlZwMkhXYXp0VlRnS2tvS2JFWjhiCmFZRzFzSU1vcEdkeUpVRVVjVllqUUhHeXpBYzhFdEppejZVdDF6TlFNYTAzL1pPS1Eyck1rSFptOFB0ZEp0RzQKeWlSRTdJUWkwcmgyZjFyREFhR1ltdS9aakg2UlFybnhyVUZzYU5GWlNvcU5oMkpodWlFSkdYQ1hhdTFUTG15Sgo2ZWY2V3FGSUFVbDVuZEsySHBOWThTbkozQUJ2U08wTVVTMmJpNVFTMXVhU3puYWhERHkxUWFOdmovN2d6cjgwCmV1cXBxcURnZzltdkFOQnhKQVJYdm1obHBBMFA4QkF4YVRtL0R5ZFNqUUNVVFgxbXFSZFc1VTdGQXJRcnJiR3oKSGRYa0FoNlZLcXlOLzhIM0NSMEJBb0lCQVFENDl4Qm40U2pEaDFqck02M0tnOUF4TDA5bjNpOXJiemZNWDdodQpaWG5OUjM5NzkyTERpZ1p6VzE2SFVyQTJCRVdSYTNra3RLN2RvUDZxWno2eDQ4N3lQQXJMVGZyZlNxYjliRmpmCm15SkFkTTBodjBUQU9GSE0wK2c3S0Y2RHVDSEEvYS9wd08weDY4UUhaN28xbHZEUE9aUkVpVlJGM2dFTWdKRjgKTlBvc1ZkV25rQ1pYbDJhUFpUZzJOSUFCMGZTeWZVL3d0c0h1amh4dEtZV0xyb3dmZ0N6bVpWc2xhcW5FM05BUApiQTlaQkJuQUhhUjEzeTdISEN5OUhGdWpCMzVyUGV5c2p0cDI0QkhwdUo3bTVyd09kbmZSU2hzcnBJdXlPdEw4CnA4cmJqemsrNWxLTVNCZmo0MUNqTmExVFlIOFJTWGN1M2dUcmVZdkJlYVkvaC8rZEFvSUJBUURFTU5LUTJNV1YKUUVVR1gyZEl4cTNUa1JSTjFJTG5leWFJZU91dFNmVkRoSUZ1WlFrM2E2NUlHbTBEbW1GRWFkS3BjNWdMZnNwNApmVWJvN1lTU3dhWEJzQlp0bjlnblJXS1FkZ1R4dzdZS3doYzBqblkrZVdYeGZNOVhQSE0za2JYUzBISFNaT3JSClhGSDNheWI1V3o0VHhwM1kvSjJtcGttbmlXMGZLdExWOVpjYUlMZ1hlWjAxUkxxZkNOZWpKVExZR1ppK3pGdDkKcGIvRzlQZFg3WVcxWUYveCtoWEU2Q2lsempscXFIRmRobkh2cDVnR0x4YzRDV1lObzIyK1FDSFNZQzg2U24zYgpMMFNNY1I0RXMzOEcvMjJWNUxJVlhvYzBUUXAwVWZEY1loVVY0MVdva01SMk9TSVRBWUM5bnZBYWZxVkk4OHNLCmVaWXlnVzh3OXJUMUFvSUJBUUQ0UGtuQ2VyVU51ZkJFbmNRRmNVZHZNNUJHcmptME16SjgrMWpINHpEL0tmS0kKNWxRNVMzQkJKL0xxbGQyVUR0QmJQc0dOZ3dmMWYybE8raUYrZVB0SmQrci9hdUxpTU9xdk9KQ3BiV05LeCt3ZQpZVHdwT2o3K01MR1lBeG15MXkvNDRqdThwWjBkTU12RzRudStvYUc5enRqek9jZW8zc05HOXcrWnZLMVM5Y2RUCkRCM2ZLdHlkMEx5cTk5QkhnRlV3Z0Zqc1dSNm9RbFUvMTY0TWFGL1pyUkdZTGFvamRlYVBuK2xwNTBLcWJMZE0KWTRJdjhma1BtaDFWOTJlNytHWHFndFZ4L2dNQmswenBNaWhuYmR4SHc0S1hVZ0FqbFMraDZKdW1SNXl6TG0xVApOWTlMeHpyakJTN0xmbU0wQnJ6TXZPYzArVFlJb2Fwam9Xdk9YMG5WQW9JQkFCVVJvU3RJL0Q4QS9laW5Takk0CmsrWktpRUdyZHJ0aE1Fd3JvRE9sNDU3eWxldkRFZkJQc2hHd05ORFVQV25aYTNRakk4cm9QTm9mcWdQTnJoVU0Ka3I1d0tKaHhPQWRQbmp3aFVIcWVKK2lUMjJZYmZudExFaldTejdsd2xuYjdRT2w0MVNCaEVnNlZ1WCsybENMbgpONDFzSVB0eWRZTzJDK2JnRFVYeGxWN0ExdzlKUUR2VkpaclkzS25EaTFUTDQ5L3RMOGdkcmgyYU5UUXFqbjEwCjFvMFo0blBjQllaMTRCZWVRL0ErVXA1V2w5bkN4OEt5UCs0V3BFMEdwZnh1YXJOcS9PZG5wSWhyVlJNMytwOUsKbjNPaTdxUFFRWWVsOVNNYXV1cXUrZ3pRdzY3c0VRRGZPeG52SE1lcHU2ZWhiK3VJZWp1Ull0YW5KQWdjZWxKcAp6QmtDZ2dFQkFQUVMxUlNTVEhaTDdZUGI0K3FiMml1T0lRbzVBY2p3dGs4a2k2RS9hazM4aEpPcnVqNjdXNEM3CmVIYWYrQVNKVmRFaUdKU3VyazBtdG80NkpKcVUwSzB5azNLRk50MzdNV2dDaWo0RXdUNmdqblZGY1ZXMTJSQVcKcE5jUHdGM2MzRFIzZWNuRnhkZGV2NG9YMllFQXlzeXBCeDlZMFZqTzY2UVhvT25sTXFEUzE3UlB0R3h0SkxEbApsd241cGZJYlVOMEtRSDRBS2p5SExPbFllMXhmV0QzUlZVQXBLVXNLMGdDdi9VQk42Y1dyUW0vWFlISFNkNFhKCm5uelAxRzE4YUlRcndqZ0JmWXorOHB6aVBMVnZVWmlxVmJrYmdmT0t3K21xb2wwYUYzNURwZEN4ckdTQ3R1ODYKWmRGWkU2SlNZRWd4OHBiQitJSVlqaDduS1hJa0Y2QT0KLS0tLS1FTkQgUFJJVkFURSBLRVktLS0tLQo=
kind: Secret
metadata:
  creationTimestamp: "2022-06-04T15:59:27Z"
  name: my-tls-secret
  namespace: default
  resourceVersion: "18994"
  uid: 799d5eba-c8cc-4603-ae95-f40dcbc7f01d
type: kubernetes.io/tls
```

- 하지만, 시크릿의 데이터가 길어지면 YAML 파일에 직접 시크릿의 데이터를 저장하는 건 바람직한 방법 아님
- 위의 YAML에서도 보면 tls.crt의 데이터가 매우 길어 가독성도 좋지 못하고 파일과 데이터를 분리하지 못해 관리하기에도 불편함

- 이러한 단점을 해결하기 위해 데이터를 YAML 파일로 부터 분리할 수 있는 kustomize 기능을 사용
- kubectl 1.14 버전부터 사용할 수 있는 기능으로 자주 사용되는 YAML 파일의 속성을 별도로 정의해 재사용하거나 여러 YAML 파일을 하나로 묶는 등 다양한 용도로 사용할 수 있는 기능
- 지금은 시크릿과 컨피그맵을 좀 더 쉽게 쓰기 위한 용도로 kustomize를 간단히 사용

```bash
# kustomization.yaml

serectGenerator:
- name: kustomize-secret
	type: kebernetes.io/tls
	files:
	- tls.crt=cert.crt
	- tls.key=cert.key
```

- 여태까지 사용한 YAML 파일과는 다른 형식으로 보이지만 시크릿을 생성하던 때 사용한 방법과 크게 다르지 않음
- kustomize 에서 시크릿 생성하기
    - `kubectl kustomize ./`
    
    ```bash
    apiVersion: v1
    data:
      tls.crt: |
        LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUVxRENDQXBBQ0NRRHUwNTVhN3JuRVNEQU5CZ2txaGtpRzl3MEJBUXNGQURBV01SUXdFZ1lEVlFRRERBdGwKZUdGdGNHeGxMbU52YlRBZUZ3MHlNakEyTURReE5UVTRNemxhRncweU16QTJNRFF4TlRVNE16bGFNQll4RkRBUwpCZ05WQkFNTUMyVjRZVzF3YkdVdVkyOXRNSUlDSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQWc4QU1JSUNDZ0tDCkFnRUF2c3lqcWlVY2U1ZjBJOFlaNWVVdDhhcHhwTWdJSEJ6VzFCdUFpSWpNM2lXN016SlZUNGRRd25QZ2JSdmoKVjZoL3g4eWhaTUtyWjlJNzIwTkhadHhIUGE2ajRpVkZvaUd1OVNvYlZWaGxvVVZNVkpFbEZBMlkxYmpKbHhUeQpPN1ViUDJiclBXMS9DQjNWNzJPckZCMTJ6Q2dVRUlqNXNWcUpuSldBL2c3emd0MXVBVDhHdnp3NkpOVzQ5YkVKCnIxNFhrYzdwc1FMckdiNVFiZGNGWHgwTUZqL2pZb3Zaci9ja09raHRUS3N4VTNKR0hRd0U5K05OcXJFNlJpS20KMTU3WGwvbExycThxdnQxbmQzNTBqYU9xWHJkcitadHcvQmJlZWgxQ1ZxeFFYaU4xdlowMGE4ZFQxd1loTnIwYwpDWmhxUGh1QVdsYXJzeng1eEEyZWxmSEJrNS9ZaVJmaFM4Z0h5TDlqbzZaNVpBaUpFeFRMSEZycVhuTERzYUk2Ck5WOUF6aEpKTEJPZnZtUHVNTFFFM0xWQUdQbUdHTUtOYmlJUU1rMnhJN3F4VUdXUDFQNi9EV3RidlZtZkFJVVQKcmlNTGprV3p4eUxOZEhUTGtGcWY3aXpZd2hLYTUwRjk4aVhYbGdIV21xdTlleGJ3enlsbUFRK2thMkpJRFdVdAorS1pma2h6NDh6L2F2MjM4ODhUMFlNQ2MwU2VBQ0w4QmN0VkJ5OElJcEY4UEZ5blhjMFpBNmRUcHdjTDVqcUNICnZONk9NY0ZNWFpUOHhjZHg2b0k4Y2lXTW5WVTZMUFlQSHpJV0ZuTlRWNTg3ZHBpQTRVUUpGZGEzbHdPTUpqanAKWmdwdUtrMS92RE5jOE1VZWc0UnNBL29mYmFITTFybFQ3UFQwMmN6UGRuM0FCVUVDQXdFQUFUQU5CZ2txaGtpRwo5dzBCQVFzRkFBT0NBZ0VBSEp5SGcxZy9uU1Y3VHBOcUVsZUZsTldlOEk4dHRWSHQvc1ZiRHhxNXdPTVFKZC81CjRyakJzMkpQZ0RhVEk4YjZyWDFpcHVEME5hREFHdTRwMzJIKzZ2cFZqT3RIclJsd3lKVzdRZ3ZXUHVleXpiU2EKcTc0NTJaV3FHeUhibjgxSEMzQ2l5SnpJZTBWTVJacEZUY3NnUDF5VmpmNTZlWVpEWEd6OU52NUwwcUJqbmNKSgp0WGQvUEgrbDd5L25jZmdWTldjenljdFNMaXB1Y2FIR3NkZ3R2N3VuUFJmSEZxNDVCZUtJOGZ4VDgxb2hoNU9hCnRaby8yNEZqYVZmajdtczRJWkJPWC9KRWEvVUZNZnZldXJwb0lRSjd3YnQxU2pHWlUzRHUvK2NyR2x1YzNTQm8KVEdrY09XMllRNC9ERTVXZ0pBSjNwOHlhcnBOc0NTTTJsV3RLWGxnbGxxVW9BdVduWFlsWkswTHRPUW91NXd3VQplbDlIelBVa09RMzY3eEUyaHRPOVkxSll1dG9ZVE4zUEc3bTExZHNPNzBpRTlMaDkxb1RaS3RPY2cwemNaSjRqCmtuNmloMGtUalhnSWUzVTE4RHJiVVE1UTJxZEpLVnp0OFVqU0FYSnRBTzZ6TG1tVzF3dGlmYjBaTDBRa2haaHUKZ3hMcjFpL3Y2b0l5dFcrUGd0c1B1TkFHUk9VeHY4SW5QbFpUZ2c1VDhqeXNucXdCMmk5Z3FXclNNTmxNKzlObgp2QWtNd2V6SnkxSGRRajJ3Y1BVS2dYMTVmTzRSMXA2Nkp6cVVBSUVBdFNmNTJoaFFnSWNtdmRtVnBsNHArbWlsCkJaUWJLWWVkTFJ4SFRhYmxMWUhkSkNpdyt2ZU52MFdmNXFzK3BZd0RXS1N3Qk1LZ2pLRmxtRzk5dUs4PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
      tls.key: |
        LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUpRd0lCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQ1Mwd2dna3BBZ0VBQW9JQ0FRQyt6S09xSlJ4N2wvUWoKeGhubDVTM3hxbkdreUFnY0hOYlVHNENJaU16ZUpic3pNbFZQaDFEQ2MrQnRHK05YcUgvSHpLRmt3cXRuMGp2YgpRMGRtM0VjOXJxUGlKVVdpSWE3MUtodFZXR1doUlV4VWtTVVVEWmpWdU1tWEZQSTd0UnMvWnVzOWJYOElIZFh2Clk2c1VIWGJNS0JRUWlQbXhXb21jbFlEK0R2T0MzVzRCUHdhL1BEb2sxYmoxc1FtdlhoZVJ6dW14QXVzWnZsQnQKMXdWZkhRd1dQK05paTltdjl5UTZTRzFNcXpGVGNrWWREQVQzNDAycXNUcEdJcWJYbnRlWCtVdXVyeXErM1dkMwpmblNObzZwZXQydjVtM0Q4RnQ1NkhVSldyRkJlSTNXOW5UUnJ4MVBYQmlFMnZSd0ptR28rRzRCYVZxdXpQSG5FCkRaNlY4Y0dUbjlpSkYrRkx5QWZJdjJPanBubGtDSWtURk1zY1d1cGVjc094b2pvMVgwRE9Fa2tzRTUrK1krNHcKdEFUY3RVQVkrWVlZd28xdUloQXlUYkVqdXJGUVpZL1UvcjhOYTF1OVdaOEFoUk91SXd1T1JiUEhJczEwZE11UQpXcC91TE5qQ0Vwcm5RWDN5SmRlV0FkYWFxNzE3RnZEUEtXWUJENlJyWWtnTlpTMzRwbCtTSFBqelA5cS9iZnp6CnhQUmd3SnpSSjRBSXZ3RnkxVUhMd2dpa1h3OFhLZGR6UmtEcDFPbkJ3dm1Pb0llODNvNHh3VXhkbFB6RngzSHEKZ2p4eUpZeWRWVG9zOWc4Zk1oWVdjMU5Ybnp0Mm1JRGhSQWtWMXJlWEE0d21PT2xtQ200cVRYKzhNMXp3eFI2RApoR3dEK2g5dG9jeld1VlBzOVBUWnpNOTJmY0FGUVFJREFRQUJBb0lDQUUwWGdrbU5GU1ViRUpvandQTVMxcTErCm9NeGp4bU1WZy9mUDVPOUYxd0VyWGFnaC9qWlVCbDJMVkhMQmdlbzVPdWdQMW1aUUFkSEJNRTQzc1BIdXJ4cE4KSmdxSjVNak5zMU43Mys5citDUmhTNllmdjB1Szh1WG45QXdIZXBpRlpLMEplS01wU3RxTXM1UTJRVG12YmdDdgpjT3Y5YkdZc25zMlYycmpNY2JlK29HUUFnMGxobkZ5bHZrWUhjbEpaUWt5M3ZkUzN5U1p0cnpHeVg0ayt3MU42CkhQWUVhOENkcXhXaGpnZ2NZNkhEMm5DQ0dyL09KK09BR0h1ZUpLdWFrcUhsS0o4OFI1azIvRWRiNE53WjlReXcKTUFCNnZmd2RnV2IxeTRnWnQ1OVIxSkQ0bytXb2RFZTlRazVMdXJobHpRZXJOUUZMWTdUWUNwc2NwYWNRTjVZcQpjaFd0SU1xdjJNMXl1NEhnRm1MNXhHeVhqOVhIeGJoR1htTG9xY25XaVhzYlZwMkhXYXp0VlRnS2tvS2JFWjhiCmFZRzFzSU1vcEdkeUpVRVVjVllqUUhHeXpBYzhFdEppejZVdDF6TlFNYTAzL1pPS1Eyck1rSFptOFB0ZEp0RzQKeWlSRTdJUWkwcmgyZjFyREFhR1ltdS9aakg2UlFybnhyVUZzYU5GWlNvcU5oMkpodWlFSkdYQ1hhdTFUTG15Sgo2ZWY2V3FGSUFVbDVuZEsySHBOWThTbkozQUJ2U08wTVVTMmJpNVFTMXVhU3puYWhERHkxUWFOdmovN2d6cjgwCmV1cXBxcURnZzltdkFOQnhKQVJYdm1obHBBMFA4QkF4YVRtL0R5ZFNqUUNVVFgxbXFSZFc1VTdGQXJRcnJiR3oKSGRYa0FoNlZLcXlOLzhIM0NSMEJBb0lCQVFENDl4Qm40U2pEaDFqck02M0tnOUF4TDA5bjNpOXJiemZNWDdodQpaWG5OUjM5NzkyTERpZ1p6VzE2SFVyQTJCRVdSYTNra3RLN2RvUDZxWno2eDQ4N3lQQXJMVGZyZlNxYjliRmpmCm15SkFkTTBodjBUQU9GSE0wK2c3S0Y2RHVDSEEvYS9wd08weDY4UUhaN28xbHZEUE9aUkVpVlJGM2dFTWdKRjgKTlBvc1ZkV25rQ1pYbDJhUFpUZzJOSUFCMGZTeWZVL3d0c0h1amh4dEtZV0xyb3dmZ0N6bVpWc2xhcW5FM05BUApiQTlaQkJuQUhhUjEzeTdISEN5OUhGdWpCMzVyUGV5c2p0cDI0QkhwdUo3bTVyd09kbmZSU2hzcnBJdXlPdEw4CnA4cmJqemsrNWxLTVNCZmo0MUNqTmExVFlIOFJTWGN1M2dUcmVZdkJlYVkvaC8rZEFvSUJBUURFTU5LUTJNV1YKUUVVR1gyZEl4cTNUa1JSTjFJTG5leWFJZU91dFNmVkRoSUZ1WlFrM2E2NUlHbTBEbW1GRWFkS3BjNWdMZnNwNApmVWJvN1lTU3dhWEJzQlp0bjlnblJXS1FkZ1R4dzdZS3doYzBqblkrZVdYeGZNOVhQSE0za2JYUzBISFNaT3JSClhGSDNheWI1V3o0VHhwM1kvSjJtcGttbmlXMGZLdExWOVpjYUlMZ1hlWjAxUkxxZkNOZWpKVExZR1ppK3pGdDkKcGIvRzlQZFg3WVcxWUYveCtoWEU2Q2lsempscXFIRmRobkh2cDVnR0x4YzRDV1lObzIyK1FDSFNZQzg2U24zYgpMMFNNY1I0RXMzOEcvMjJWNUxJVlhvYzBUUXAwVWZEY1loVVY0MVdva01SMk9TSVRBWUM5bnZBYWZxVkk4OHNLCmVaWXlnVzh3OXJUMUFvSUJBUUQ0UGtuQ2VyVU51ZkJFbmNRRmNVZHZNNUJHcmptME16SjgrMWpINHpEL0tmS0kKNWxRNVMzQkJKL0xxbGQyVUR0QmJQc0dOZ3dmMWYybE8raUYrZVB0SmQrci9hdUxpTU9xdk9KQ3BiV05LeCt3ZQpZVHdwT2o3K01MR1lBeG15MXkvNDRqdThwWjBkTU12RzRudStvYUc5enRqek9jZW8zc05HOXcrWnZLMVM5Y2RUCkRCM2ZLdHlkMEx5cTk5QkhnRlV3Z0Zqc1dSNm9RbFUvMTY0TWFGL1pyUkdZTGFvamRlYVBuK2xwNTBLcWJMZE0KWTRJdjhma1BtaDFWOTJlNytHWHFndFZ4L2dNQmswenBNaWhuYmR4SHc0S1hVZ0FqbFMraDZKdW1SNXl6TG0xVApOWTlMeHpyakJTN0xmbU0wQnJ6TXZPYzArVFlJb2Fwam9Xdk9YMG5WQW9JQkFCVVJvU3RJL0Q4QS9laW5Takk0CmsrWktpRUdyZHJ0aE1Fd3JvRE9sNDU3eWxldkRFZkJQc2hHd05ORFVQV25aYTNRakk4cm9QTm9mcWdQTnJoVU0Ka3I1d0tKaHhPQWRQbmp3aFVIcWVKK2lUMjJZYmZudExFaldTejdsd2xuYjdRT2w0MVNCaEVnNlZ1WCsybENMbgpONDFzSVB0eWRZTzJDK2JnRFVYeGxWN0ExdzlKUUR2VkpaclkzS25EaTFUTDQ5L3RMOGdkcmgyYU5UUXFqbjEwCjFvMFo0blBjQllaMTRCZWVRL0ErVXA1V2w5bkN4OEt5UCs0V3BFMEdwZnh1YXJOcS9PZG5wSWhyVlJNMytwOUsKbjNPaTdxUFFRWWVsOVNNYXV1cXUrZ3pRdzY3c0VRRGZPeG52SE1lcHU2ZWhiK3VJZWp1Ull0YW5KQWdjZWxKcAp6QmtDZ2dFQkFQUVMxUlNTVEhaTDdZUGI0K3FiMml1T0lRbzVBY2p3dGs4a2k2RS9hazM4aEpPcnVqNjdXNEM3CmVIYWYrQVNKVmRFaUdKU3VyazBtdG80NkpKcVUwSzB5azNLRk50MzdNV2dDaWo0RXdUNmdqblZGY1ZXMTJSQVcKcE5jUHdGM2MzRFIzZWNuRnhkZGV2NG9YMllFQXlzeXBCeDlZMFZqTzY2UVhvT25sTXFEUzE3UlB0R3h0SkxEbApsd241cGZJYlVOMEtRSDRBS2p5SExPbFllMXhmV0QzUlZVQXBLVXNLMGdDdi9VQk42Y1dyUW0vWFlISFNkNFhKCm5uelAxRzE4YUlRcndqZ0JmWXorOHB6aVBMVnZVWmlxVmJrYmdmT0t3K21xb2wwYUYzNURwZEN4ckdTQ3R1ODYKWmRGWkU2SlNZRWd4OHBiQitJSVlqaDduS1hJa0Y2QT0KLS0tLS1FTkQgUFJJVkFURSBLRVktLS0tLQo=
    kind: Secret
    metadata:
      name: kustomize-secret-48h68k476c
    type: kebernetes.io/tls
    ```
    

![Untitled](../../../assets/images/container/kubenetes/config_secret/Untitled%2017.png)

- kustomizer 를 통해서 컨피그맵 만들기

```bash
# kustomization.yaml

configMapGenerator:
- name: kustomize-configmap
	files:
	- tls.crt=cert.crt
	- tls.key=cert.key
```

- 시크릿맵과 비슷한데 type이 없으므로 type만 제거하고 작성하고 정의하면 됨

```bash
🗒️ Note!
kustomize로부터 생성된 컨피그맵이나 시크릿의 이름 뒤에는 추출된 해시값이 자동으로 추가됨.
kubectl 명령어로 생성할 때에도 해쉬 값을 추가하고 싶다면 --append-hash 를 추가로 붙이면 가능

kubectl create secret tls kustomize-secret \
--cert cert.crt --key cert.key --append-hash
```

### 컨피그맵이나 시크릿을 업데이트해 애플리케이션의 설정값 변경하기

1. 컨피그맵이나 시크릿의 값을 kubectl edit 명령어로 수정하기
2. YAML 파일을 변경한 뒤 다시 kubectl apply 명령어를 사용해서 적용하기
3. kubectl patch 명령어 사용하기

### 정리하며…

- 컨피그맵과 시크릿을  포드에 제공하는 방법을 크게 두 가지 설명
    1. 환경 변수로 포드 내부에 설정값을 제공
    2. 볼륨 파일로 포드 내부에 마운트하는 방법
- 1번 방법은 컨피그맵이나 시크릿의 값을 변경해도 자동으로 재설정되지 않으며, 디플로이먼트의 포드를 다시 생성해야함.
- 2번 방법은 컨피그맵이나 시크릿을 변경하면 파일의 내용 또한 자동으로 갱신됨. 그러나 실행중인 애플리케이션에 적용되는 것은 별개의 이야기 적용하려면 별도의 로직을 개발자가 구성해야함
    - 예를 들면, 변경된 파일을 다시 읽어 들이도록 프로세스에 별도의 시그널을 보내는 사이드카 컨테이너를 포함시키기
    - 애플리케이션의 소스코드 레벨에서 쿠버네티스의 API를 통해 컨피그맵, 시크릿의 데이터 변경에 대한 알림을 받은 뒤, 자동으로 리로드 하는 로직

- 리소스 정리

```bash
kubectl delete deployment --all
kubectl delete pod --all
kubectl delete configmap --all
kubectl delete secret --all
```