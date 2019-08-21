# MicroK8S 설정

## Install MicroK8S

- 참고: https://microk8s.io/
 
- `snap`을 이용한 설치 명령어
```bash
$ sudo snap install microk8s --classic
```

### Tips

- 더 많은 정보는 다음을 참고할 것
    
    - https://microk8s.io/docs/

- Enable GPU

    - NVIDIA를 기준으로
    ```bash
    # 먼저 nvidia-smi를 입력하여
    # 그래픽카드 드라이버가 설치되어 있는지 확인할 것.
    $ nvidia-smi

    Wed Aug 21 11:19:46 2019       
    +-----------------------------------------------------------------------------+
    | NVIDIA-SMI 430.26       Driver Version: 430.26       CUDA Version: 10.2     |
    |-------------------------------+----------------------+----------------------+
    | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
    |===============================+======================+======================|
    |   0                      Off  | 00000000:01:00.0  On |                  N/A |
    | N/A   35C    P8     2W /  N/A |    244MiB /  3911MiB |      2%      Default |
    +-------------------------------+----------------------+----------------------+
                                                                               
    +-----------------------------------------------------------------------------+
    | Processes:                                                       GPU Memory |
    |  GPU       PID   Type   Process name                             Usage      |
    |=============================================================================|
    |    0      7247      G   /usr/lib/xorg/Xorg                           100MiB |
    |    0      7410      G   /usr/bin/gnome-shell                         142MiB |
    +-----------------------------------------------------------------------------+

    $ microk8s.enable gpu

    # 이후 microk8s.kubectl describe nodes 명령을 입력하여 
    # Capacity 부분과 Allocatable 부분에 "nvidia.com/gpu" 부분이 등록되어 있는지
    # 확인

    $ microk8s.kubectl describe nodes
    
    ...
    Capacity:
        cpu:                8
        ephemeral-storage:  244568380Ki
        hugepages-1Gi:      0
        hugepages-2Mi:      0
        memory:             16269480Ki
        nvidia.com/gpu:     1
        pods:               110
    Allocatable:
        cpu:                8
        ephemeral-storage:  243519804Ki
        hugepages-1Gi:      0
        hugepages-2Mi:      0
        memory:             16167080Ki
        nvidia.com/gpu:     1
        pods:               110
    ...
    ```
## Install Python API

- 참고: https://github.com/kubernetes-client/python

- `pip`명령어를 터미널에 입력하여 각 `pip`가 어디를 보고 있는지 확인과정을 거쳐야 함.
 ```bash
 $ pip3 -V
 pip 9.0.1 from /usr/lib/python3/dist-packages (python 3.6)
 ```
- 만약에 `pip`가 설치되지 않았다면 우분투를 기준으로 다음 명령어를 이용하여 설치할 수 있음.
```bash
# python3 전용
$ sudo apt install python3-pip

$ pip3 -V
pip 9.0.1 from /usr/lib/python3/dist-packages (python 3.6)
```
- 이후 설치를 진행하여 `kubernetes` 패키지를 설치
```bash
# pip3 명렁어가 Python3 를 바라보고 있다는 전제 하에
$ pip3 install kubernetes
```

## Creating Configuration file

- MicroK8S를 설치하면 설정파일을 `~/.kube/` 폴더에 만들어주지 않기 때문에 직접 만들어 주어야 함. (만들어주지 않으면 파이썬에서 **에러**를 출력함.)
```bash
# 폴더가 없다면 mkdir 명령을 사용하여 폴더를 생성해준다.
$ mkdir $HOME/.kube/
$ microk8s.kubectl config view --raw > $HOME/.kube/config
```

## 예제

- 다음의 예제는 파이썬에서 앞서 설치한 `kubernetes`모듈을 통해서 배포된 쿠버네티스의 팟(Pod)을 불러오는 예제이다.
- 참고로 `ret.items`를 통해서 불러올 수 있는데, 이 부분은 `json`형식이 아닌 `class` 형식이기 때문에 items.status 와 같이 점(`.`)을 사용하여 불러와야 함.
```python3
from pprint import pprint
import json

config.load_kube_config()

v1 = client.CoreV1Api()
ret = v1.list_pod_for_all_namespaces()
for item in ret.items:
    print(item)
```
