# 쿠버네티스 설치 삽질로그

## Kubernetes Master Node Configuration

1. 초반에는 이거 따라 설정하면 끝. (apt-get 명령)

  - https://kublr.com/blog/how-to-install-a-single-master-kubernetes-k8s-cluster/
  
  - 반드시 `sudo swapoff -a` 명령을 내려서 스왑 중지할 것
  
  - 이후 `sudo kubeadm init --node-name master` 명령 내려서 마스터 노드 설정 진행 (몇분 걸림)

2. `kubectl`에서 설정파일 읽어들일 수 있도록 설정
```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

3. `sudo kubectl get nodes` 명령 실행
```bash
$ sudo kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   15m   v1.15.3
```

4. `STATUS` 부분이 `NotReady`인 경우 다음과 같이 명령
- 참고: https://stackoverflow.com/questions/53040663/kubectl-get-nodes-shows-notready
```bash
$ sudo kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(sudo kubectl version | base64 | tr -d '\n')"
```

5. 몇분 후 `sudo kubectl get nodes` 명령 실행하여 Ready 상태인 것을 확인
```bash
# 다음 명령을 기록해 두고 다른 서버에서 설치했을 경우 노드로 편입할 때 사용.
$ kubeadm join x.x.x.x:xxxx --token xxxx.xxxxxxxxxxxxx \
    --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Kubernetes Worker Node Configureation

1. 설치 단계에서 init이 성공적으로 완료되었을 경우 **5번**과 같이 명령문구를 알려주는데 이를 복사해서 보관하고 있어야 함.
```bash
# 다음과 같이 실행해도 Join 하는 명령어를 얻을 수 있음
$ kubeadm token create --print-join-command
```

2. 각 서버에도 처음에 했던 것들을 따라서 설치한 후 `kubectl`과 `kubeadm`을 설치

3. sudo 권한을 통해서 서버에 등록하는 과정이 필요한데, 여기에 5번에 있는 부분을 읽고 복사/붙여넣기

## Kubernetes Re-Naming 

- K8S를 설치하면 다음과 같이 마스터가 나오게 되고 등록한 노드들이 리스트 형태로 나오게 되는데, `ROLES`부분을 보면 `<none>`이라고 나오는 것을 볼 수 있다.

- 이 부분을 `kubectl label` 명령어를 사용해서 바꾸어 보자
```bash
$ sudo kubectl get nodes
NAME        STATUS   ROLES    AGE   VERSION
master      Ready    master   73m   v1.15.3
ubuntu      Ready    <none>   23m   v1.15.3
ubuntu114   Ready    <none>   14s   v1.15.3
```

- 라벨은 다음과 같이 `kubectl label` 명령어를 사용하여 변경할 수 있다.
```bash
$ sudo kubectl label node ubuntu node-role.kubernetes.io/worker=worker
# node/ubuntu labeled
$ sudo kubectl label node ubuntu114 node-role.kubernetes.io/worker=worker
# node/ubuntu114 labeled
```

- 변경한 후 다시 입력해보면...
```bash
$ sudo kubectl get nodes
NAME        STATUS   ROLES    AGE     VERSION
master      Ready    master   79m     v1.15.3
ubuntu      Ready    worker   29m     v1.15.3
ubuntu114   Ready    worker   6m29s   v1.15.3
```

## Integrate With NVIDIA-GPU

- GPU와 연동하는 방법을 알아보자

	- 여기에서는 NVIDIA만 사용했지만, AMD와도 사용하여 연동할 수 있다.

	- `AMD`는 ROCM을 사용할 수 있다. (사이트를 보아하니 지원한다고는 하는 것 같은데 잘 모르겠음.)

### 설치전 꼭! 확인해야 할 것

1. 드라이버가 깔려있는지 확인

	- `cat /proc/driver/nvidia/version` 확인

	- 없으면 **NVIDIA 드라이버 먼저 설치**

2. `nvidia-docker`이 깔려있는지 확인

	- `nvidia-docker`을 터미널에서 입력하는 것 만으로 설치 확인 가능.

	- 없으면 **NVIDIA-DOCKER** 설치

3. 이후 `nvidia-smi`입력하여 GPU가 정상적으로 뜨는지 확인
```bash
$ nvidia-smi
Tue Aug 20 16:43:20 2019       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 384.130                Driver Version: 384.130                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0                      On   | 00000000:3B:00.0 Off |                    0 |
| N/A   40C    P0    26W / 250W |      0MiB / 16276MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+

```

### Docker-Daemon 설정

1. 다음 문서 참고하여 설정 (**마스터가 아니라 각 노드에 설정**)

	- https://likefree.tistory.com/15

2. 그래픽 드라이버가 이미 설치되어 있는 경우라면 다음만 설정만 해주면 됨.

	- `/etc/docker/daemon.json`파일 확인하여 디폴트 런타임이 NVIDIA인지 확인
	```json
	{
	    "default-runtime": "nvidia",
		"runtimes": {
			"nvidia": {
				"path": "nvidia-container-runtime",
	            "runtimeArgs": []
	        }
	   	}
	}
	```

	- `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` 파일 확인 및 `SERVICE`부분에 다음 줄 추가한 후 저장
	```bash
	Environment="KUBELET_EXTRA_ARGS=--feature-gates=DevicePlugins=true"
	```

	- `데몬 적용`
	```bash
	$ systemctl daemon-reload
	
	$ systemctl restart kubelet
	```

### Install NVIDIA Plugin

- 다음 명령어를 입력하여 NVIDIA 플러그인을 연결된 노드에 한번에 배포할 수 있다.
```bash
$ kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta/nvidia-device-plugin.yml
```

- 이후 `kubectl get pods --all-namespaces`를 입력했을 때 `nvidia-device-plugin-daemonset`이 올라오면 완료된 것이다.
```bash
$ kubectl get pods --all-namespace
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE
kube-system   coredns-5c98db65d4-jtk5q               1/1     Running   0          143m
kube-system   coredns-5c98db65d4-ms9qm               1/1     Running   0          143m
kube-system   etcd-master                            1/1     Running   0          142m
kube-system   kube-apiserver-master                  1/1     Running   0          142m
kube-system   kube-controller-manager-master         1/1     Running   0          142m
kube-system   kube-proxy-42v6f                       1/1     Running   0          143m
kube-system   kube-proxy-hrw4w                       1/1     Running   0          93m
kube-system   kube-proxy-lq5zf                       1/1     Running   1          70m
kube-system   kube-scheduler-master                  1/1     Running   0          142m
kube-system   nvidia-device-plugin-daemonset-2rbfl   1/1     Running   0          12m
kube-system   nvidia-device-plugin-daemonset-g95kv   1/1     Running   0          14m
kube-system   weave-net-r98k8                        2/2     Running   0          135m
kube-system   weave-net-wvjd2                        2/2     Running   0          93m
kube-system   weave-net-x9nlj                        2/2     Running   3          70m
```

## GPU 갯수 Counting

- 앞서 정리한 것들이 성공적으로 동작하였다는 전제 하에, `kubectl describe nodes`를 입력했을 경우에 다음과 같이 `Allocatable` 항목에 GPU 들이 뜨는 것을 확인할 수 있다.
```bash
$ kubectl describe nodes

...
Allocatable:
cpu:                32
ephemeral-storage:  395253912536
hugepages-1Gi:      0
hugepages-2Mi:      0
memory:             15965916Ki
nvidia.com/gpu:     1
pods:               110

...

# Allocated Resources는 사용하고 있는 리소스 정보를 보여주는데, 여기에 사용하고 있는 GPU의 갯수를 알려준다.
Allocated resources:
(Total limits may be over 100 percent, i.e., overcommitted.)
Resource           Requests  Limits
--------           --------  ------
cpu                20m (0%)  0 (0%)
memory             0 (0%)    0 (0%)
ephemeral-storage  0 (0%)    0 (0%)
nvidia.com/gpu     0         0

...
```
