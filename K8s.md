## 쿠버네티스 환경 구축
###  KVM 설치
- **KVM : 물리적 리눅스 시스템에 가상 머신을 생성할 수 있는 소프트웨어 기능 (즉 KVM을 사용하면 서버 관리자가 가상화 인프라를 수동으로 프로비저닝할 필요가 없으며 클라우드 환경에 많은 수의 가상머신을 손쉽게 배포가 가능해짐)**
- 참조 : https://aws.amazon.com/ko/what-is/kvm/
- 여기서부턴 ssh로 접속 후 진행
#### 1.  가상화 지원 여부 확인
>  grep -E '(vmx|xvm)' /proc/cpuinfo  
> lscpu | grep Virtualization
```
Virtualization:                  VT-x
```

#### 2. KVM 설치 진행
> sudo dnf -y install qemu-kvm libvirt virt-manager virt-install  
> sudo dnf -y install epel-release  
> sudo dnf -y install bridge-utils virt-top libguestfs-tools bridge-utils virt-viewer  

#### 3. 설치 확인
> lsmod | grep kvm

---
	kvm_intel             479232  0
	kvm                  1327104  1 kvm_intel
	irqbypass              16384  1 kvm
---
- 시스템 재기동 및 확인
> sudo systemctl restart libvirtd  
> sudo systemctl enable libvirtd  
> sudo systemctl status libvirtd
 - Active: active (running) #active(running)가 정상 동작중인 상태
 - 작업 실패: failed to read the PCI VPD data 오류가 뜸
```
[insight@localhost ~]$ sudo systemctl status libvirtd
● libvirtd.service - libvirt legacy monolithic daemon
     Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; enabled; preset: disabled)
     Active: active (running) since Fri 2024-05-17 16:37:06 KST; 22s ago
TriggeredBy: ● libvirtd.socket
             ● libvirtd-admin.socket
             ● libvirtd-ro.socket
       Docs: man:libvirtd(8)
             https://libvirt.org/
   Main PID: 11532 (libvirtd)
      Tasks: 21 (limit: 32768)
     Memory: 25.3M
        CPU: 370ms
     CGroup: /system.slice/libvirtd.service
             ├─11532 /usr/sbin/libvirtd --timeout 120
             ├─11631 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/>
             └─11632 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/>

 5월 17 16:37:06 localhost.localdomain dnsmasq-dhcp[11631]: DHCP, sockets bound exclusively to interface virbr0
 5월 17 16:37:06 localhost.localdomain dnsmasq[11631]: reading /etc/resolv.conf
 5월 17 16:37:06 localhost.localdomain dnsmasq[11631]: using nameserver 1.214.68.2#53
 5월 17 16:37:06 localhost.localdomain dnsmasq[11631]: using nameserver 61.41.153.2#53
 5월 17 16:37:06 localhost.localdomain dnsmasq[11631]: read /etc/hosts - 2 addresses
 5월 17 16:37:06 localhost.localdomain dnsmasq[11631]: read /var/lib/libvirt/dnsmasq/default.addnhosts - 0 addresses
 5월 17 16:37:06 localhost.localdomain dnsmasq-dhcp[11631]: read /var/lib/libvirt/dnsmasq/default.hostsfile
 5월 17 16:37:06 localhost.localdomain libvirtd[11532]: libvirt version: 10.0.0, package: 6.2.el9_4 (Rocky Linux Build >
 5월 17 16:37:06 localhost.localdomain libvirtd[11532]: hostname: localhost.localdomain
 5월 17 16:37:06 localhost.localdomain libvirtd[11532]: 작업 실패: failed to read the PCI VPD data
```

### 네트워크 브릿지 구성
- **네트워크 브릿지 : vm 호스트의 네트워크와 게스트의 네트워크를 연결하여 네트워킹 하는 방식 (즉 호스트와 게스트를 하나로 연결하여 두 개의 네트워크를 마치 하나의 네트워크 처럼 사용하기 위함)**
- 참조 : https://access.redhat.com/documentation/ko-kr/red_hat_enterprise_linux/9/html/configuring_and_managing_networking/configuring-a-network-bridge_configuring-and-managing-networking

> sudo nano /home/insight/create_bridge.sh
 - 아래 내용 입력 후 저장

``` /home/insight/create_bridge.sh 파일
# delete network bridge br0 : 기존에 있으면 삭제하고 시작
sudo nmcli connection delete br0

#BR_NAME: 생성할 브릿지의 이름입니다.
#BR_INT: 브리지 슬레이브로 사용될 물리적 네트워크 장치입니다.
#SUBNET_IP: 생성된 브리지에 할당된 IP 주소 및 서브넷입니다.
#GW: 기본 게이트웨이의 IP 주소
#DNS1 및 DNS2: 사용할 DNS 서버의 IP 주소입니다.

BR_NAME="br0"
BR_INT="enp2s0"
SUBNET_IP="192.168.10.222/24"
GW="192.168.10.1"
DNS1="1.214.68.2"
DNS2="61.41.153.2"
		
# 브릿지 네트워크 정의
sudo nmcli connection add type bridge autoconnect yes con-name ${BR_NAME} ifname ${BR_NAME}
#브릿지에 IP, GateWay, Dns 추가
sudo nmcli connection modify ${BR_NAME} ipv4.addresses ${SUBNET_IP} ipv4.method manual
sudo nmcli connection modify ${BR_NAME} ipv4.gateway ${GW}
sudo nmcli connection modify ${BR_NAME} ipv4.dns ${DNS1} +ipv4.dns ${DNS2}
# 식별된 네트워크 장치를 브릿지의 슬레이브로 추가
sudo nmcli connection delete ${BR_INT}
sudo nmcli connection add type bridge-slave autoconnect yes con-name ${BR_INT} ifname ${BR_INT} master ${BR_NAME}
# 네트워크 브릿지 시작
sudo nmcli connection up br0
```

- **IP를 222로 바로 변경한 경우는 없어서 정상 동작 알 수 없으나 시행 필요**
- 테스트 시 192.168.10.190/24로 변경 한 이후에 ~222/24로 변경 함

- 755로 변경
> sudo chmod 755 /home/insight/create_bridge.sh

- 파일 실행 (**본장비 리눅스PC 에서 진행**)
> /home/insight/create_bridge.sh

- 확인
> sudo nmcli connection show  
```
[insight@localhost ~]$ sudo nmcli connection show
NAME    UUID                                  TYPE      DEVICE
br0     e9dae974-5112-4e92-b87d-7f8eb9ceda82  bridge    br0
enp2s0  7f55df85-8ee8-468d-8426-3b8748d21587  ethernet  enp2s0
lo      037e2bfe-5b44-49aa-a397-aa948dff8934  loopback  lo
virbr0  4875be38-3fd2-4786-bd73-3fcc5e4bf43e  bridge    virbr0
```
> sudo nmcli connection show br0  
> ip ad  
- 192.168.10.222 로 설정된 br0 네트워크가 목록에 확인되어야 함

> sudo nano /etc/qemu-kvm/bridge.conf
```
allow all  (기존: allow virbr0)
```

> sudo systemctl restart libvirtd  
> brctl show
```
[insight@localhost ~]$ brctl show  
bridge name     bridge id               STP enabled     interfaces
br0             8000.d0509995b29f       yes             enp2s0
virbr0          8000.5254007e7428       yes
```

### VM(가상머신) 생성
1. 저장소 생성 (*700G의 /data 디렉토리에 저장소 생성*)
> su -  
> mkdir -p /data/kvm/images  
> mkdir -p /data/kvm/scripts  
> mkdir -p /data/k8s  
> mkdir -p /data/glusterfs  
> chown -R insight:insight /data  
> sudo dnf install -y guestfs-tools  

#### ubuntu 20.04 버전
2. iso 파일 다운로드
> cd /data/kvm/scripts 
> wget https://releases.ubuntu.com/focal/ubuntu-20.04.6-live-server-amd64.iso -O ubuntu-2004.iso 
> 참고 https://www.wpdiaries.com/ubuntu-on-kvm/

3. iso 이미지를 통해 vm 생성 및 qcow2 이미지 디스크 생성
>  sudo virt-install --name kvm_kube-master \
--os-variant ubuntu20.04 \
--vcpus 2 \
--memory 4096 \
--location /data/kvm/scripts/ubuntu-2004.iso,kernel=casper/vmlinuz,initrd=casper/initrd \
--network bridge=br0,model=virtio \
--disk path=/data/kvm/images/kvm_kube-master.qcow2,format=qcow2,size=30 \
--graphics none \
--extra-args='console=ttyS0,115200n8 --- console=ttyS0,115200n8' \
--debug

- vm 접속 및 우분투 설치 진행
- 설치관련 내용은 따로 없음 (대부분 엔터만 누르면 진행됨, 유저 insight / insight 로 지정)

-> 설치방법 참고 : https://velog.io/@dailylifecoding/installing-ubuntu-server-on-virtual-box\
    Ubuntu OS VM 설치하는 항목 참고
- rich mode / basic mode 중 rich mode로 진행
- 설치 중 unmounting /cdrom 에러 시 > 엔터
- SSH KEY 까지 화면에 표시됨 > 엔터 후 insight/insight로 로그인 

#### (참고) vm 관련 명령어
- VM 확인
> sudo virsh list --all
- VM 접속
> sudo virsh console (VM명)
- VM 종료
> sudo virsh shutdown (VM명)
- VM reboot
> sudo virsh reboot (VM명)
- VM start
> sudo virsh start (VM명)
- 강제종료
> sudo virsh destroy (VM명)
- 자동실행 설정
> sudo virsh autostart (VM명)
- 자동실행 목록
> sudo virsh list --autostart
- 게스트 삭제
> sudo virsh undefine (VM명)
- 볼륨 위치 확인
> sudo virsh domblklist kube-node1

#### (참고) ubuntu-18.04 버전 설치 시
2. 이미지 스크립트 생성 (총 3번 진행 (master, node1, node2))
> nano /data/kvm/scripts/kvm_kube-master.sh
```
virt-builder ubuntu-18.04 \
--format qcow2 \
-o /data/kvm/images/kube-master.qcow2 \
--size 30G \
--root-password locked:disabled \
--firstboot-command 'sudo systemctl enable serial-getty@ttyS0.service' \
--firstboot-command 'sudo systemctl start serial-getty@ttyS0.service' \
--firstboot-command 'sudo useradd -m -p "" insight -s /bin/bash; chage -d 0 insight' \
--firstboot-command 'sudo usermod -aG sudo insight' \
--firstboot-command 'sudo hostnamectl set-hostname kube-master' \
--firstboot-command 'sudo sed -i "s/ens2/enp1s0/g" /etc/netplan/01-netcfg.yaml' \
--firstboot-command 'sudo rm -f /etc/machine-id' \
--firstboot-command 'sudo systemd-machine-id-setup' \
--firstboot-command 'sudo sed -i "s/us.archive.ubuntu.com/mirror.kakao.com/g"
/etc/apt/sources.list' \
--firstboot-command 'sudo sed -i "s/security.ubuntu.com/mirror.kakao.com/g"
/etc/apt/sources.list' \
--firstboot-command 'sudo reboot'
```
- 위 내용을 복사해서 kvm_kube-master.sh 에 입력
- 전체 VM(master, node1, node2) 내용이 동일하며 생성될 이미지명 및 set-hostname 부분만 각각 *kube-master, kube-node1, kube-node2*로 변경 필요
- 앞에 공백 있으면  제거 후 ctrl+x, enter 눌러서 저장 후 권한 수정

> chmod 755 /data/kvm/scripts/kvm_kube-master.sh

- 파일 실행
> ./data/kvm/scripts/kvm_kube-master.sh

- 다운로드 완료 후 vm 생성 명령어 입력
> virt-install --name kube-master --disk path=/data/kvm/images/kube-master.qcow2 --import --vcpus 2 --ram 4096 --os-type linux --os-variant ubuntu18.04 --network bridge=br0 --graphics none

> virt-install --name kube-node1 --disk path=/data/kvm/images/kube-node1.qcow2 --import --vcpus 2 --ram 4096 --os-type linux --os-variant ubuntu18.04 --network bridge=br0 --graphics none

> virt-install --name kube-node2 --disk path=/data/kvm/images/kube-node2.qcow2 --import --vcpus 2 --ram 4096 --os-type linux --os-variant ubuntu18.04 --network bridge=br0 --graphics none

- vm 접속 후 ID, PW, PW 입력
> insight  
> insight  
> insight  

### VM 네트워크 설정
#### VM 접속
- vm 목록 확인
> sudo virsh list --all

- vm 접속 명령어
> virsh console kube-master  

- 아이피 정보 확인
> ip ad  
> sudo nano /etc/netplan/00-installer-config.yaml 
> 또는 (*우분투 버전마다 파일이름이 다름*)
> sudo nano /etc/netplan/01-netcfg.yaml
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0: //ip addr해서 나오는 명칭으로 진행 
      dhcp4: no
      addresses: [192.168.10.223/24] //vm별로 다르게 설정
      gateway4: 192.168.10.1
      nameservers:
        addresses: [1.214.68.2, 61.41.153.2] //동일하게 설정
```
- *들여쓰기 주의(띄어쓰기 2번임)*
- addresses: 아래 ip로 입력
  - kube-master: [192.168.10.223/24] 
  - kube-noe1: [192.168.10.224/24]
  - kube-node2: [192.168.10.225/24]
- host의 ip addr 했을때 나오는 inet 부분을 확인하여 ethernets 명을 적는다.
  - 예를 들어, 2: enp1s0..이면 enp1s0 적어주기

- 적용
> sudo netplan apply

- 테스트 (**다운로드가 정상적으로 되어야 함**)
> sudo apt update

- 가상머신 종료
> exit  
> ctrl + ]  

- 리눅스 재부팅 시 자동실행 설정
> sudo virsh autostart kube-master

 ->  *(참고)우분투 18.04 버전일 경우 위의 네트워크 설정과정을 master, node1, node2에 각각 해줘야 함*  (20.04 버전은 도커, 쿠버네티스 설치 이후 진행하므로 도커 설치 진행 ▶ )

#### (참고) 기존 vm 삭제 및 재생성 필요시 진행 (*root로 진행*, kube-node1일 경우)
> su -  
> virsh shutdown kube-node1  
> virsh undefine kube-node1  
> virsh list --all  
> sudo virsh domblklist kube-node1 #볼륨 정보(위치) 확인  
> sudo rm -f /data/kvm/images/kube-node1.qcow2  
> cd /data/kvm/scripts  
> ./kvm_kube-node1.sh  
> virt-install --name kube-node1 --disk path=/data/kvm/images/kube-node1.qcow2 --import --vcpus 2 --ram 4096 --os-type linux --os-variant ubuntu18.04 --network bridge=br0 --graphics none  

### 도커 설치
- 참조 : https://joyhong-91.tistory.com/50
- 쿠버네티스 : 컨테이너를 분산 배치, 상태관리 및 구동환경을 관리해주는 도구
- 도커 : 컨테이너를 다루는 도구
- **즉, 쿠버네티스는 컨테이너를 사용하는 환경이므로 컨테이너를 다루는 도구인 도커가 필요함**

#### VM 접속
- vm 목록 확인
> sudo virsh list --all

- vm 접속 명령어
> sudo virsh console kube-master   

- 로그인
> insight  
> insight

- 방화벽 비활성화
> sudo ufw disable
```
insight@kube-master:~$ sudo ufw disable
Firewall stopped and disabled on system startup
```

- 패키지 도구 업데이트
> sudo apt update && sudo apt-get update -y

- 기존 설치 삭제
> sudo apt-get remove docker docker-engine docker.io

- 도커 설치를 위한 각종 라이브러리 설치
> sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common

- key 설정
> curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
> sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
> sudo apt-get update

- 설치
> sudo apt-get install -y docker-ce docker-ce-cli containerd.io
> sudo systemctl start docker && sudo systemctl enable docker
> sudo docker version

- Docker 설치 완료 후 테스트로 hello-world 컨테이너 구동(테스트)
> sudo docker run hello-world

- 현재 도커의 group 확인 및 systemd로 변경
> sudo docker info | grep -i cgroup
```
Cgroup Driver: cgroupfs
Cgroup Version: 1
```

> 아래 명령어 한번에 입력
```
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"], 
  "log-driver":"json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

> sudo mkdir -p /etc/systemd/system/docker.service.d
> sudo systemctl daemon-reload && sudo systemctl restart docker

- 아까와 다르게 systemd라고 구성된 것 확인
> sudo docker info | grep -i cgroup
```
 Cgroup Driver: systemd
 Cgroup Version: 1
 ```

- 도커 접근 권한 수정
> sudo usermod -a -G docker $USER
> sudo systemctl restart docker
> sudo reboot

- 재기동후 insight 계정에서 docker 명령어 사용 가능
> docker

//*(참고)우분투 18.04 버전일 경우 위의 도커 설치과정을 master, node1, node2에 각각 해줘야 함*

### 쿠버네티스 (K8s) 설치
- **쿠버네티스(K8s) : 컨테이너화 된 애플리케이션을 배포, 관리, 확장할 때 수반되는 다수의 수동 프로세스를 자동화하는 오픈소스 컨테이너 오케스트레이션 플랫폼**
- 참조 : https://kubernetes.io/ko/docs/concepts/overview/

#### VM 접속
- vm 목록 확인
> sudo virsh list --all

- vm 접속 명령어
> sudo virsh console kube-master

- 스왑 메모리 비활성화
- **이유 : 쿠버네티스는 Pod을 생성 시, 필요한 만큼의 리소스를 할당 받아서 사용하는 구조이므로 메모리 Swap을 고려하지 않고 설계되었기 때문에, 쿠버네티스 클러스터 Node들은 모두 Swap 메모리를 비활성화 해줘야 함**
- 참조 : https://kgw7401.tistory.com/50
> sudo swapoff -a
> sudo swapoff -a && sudo sed -i '/swap/s/^/#/' /etc/fstab

##### (참고) 스왑 메모리 비활성화 (ubuntu 18.04 버전)
> sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

- 확인 (swap 관련 마지막줄 내용이 '#'으로 주석처리 되어있는지 확인 필요 )
> cat /etc/fstab
```실행 결과
insight@kube-master:~$ cat /etc/fstab

# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/vda1 during installation
UUID=8a5c65a8-9852-4c2a-9789-627cb736abe5 /               ext4    errors=remount-ro 0       1
#/swapfile                                 none            swap    sw              0       0
```

- 내용 추가 (아래 명령어 한번에 입력)
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

- 설치 준비
> sudo sysctl --system
> sudo apt-get install -y apt-transport-https ca-certificates curl gpg
> sudo mkdir -p  /etc/apt/keyrings
> sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
> sudo echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

- 설치
> sudo apt-get update
> sudo apt-get install -y kubelet kubeadm kubectl
> sudo apt-mark hold kubelet kubeadm kubectl

- 실행 결과
```
insight@kube-master:~$ sudo apt-mark hold kubelet kubeadm kubectl

kubelet set on hold.
kubeadm set on hold.
kubectl set on hold.
```
> sudo systemctl start kubelet
> sudo systemctl enable kubelet
- 기능 설명
  - **kubeadm** :  클러스터를 구성하기 위한 다양한 기능 제공
  - **kubelet** :  클러스터의 모든 머신에서 실행되는 Pod와 Container 시작과 같은 작업을 수행하는 구성 요소
  - **kubectl** :  클러스터와 통신하기 위한 Command-Line 유틸리티

- 설치 확인
> sudo kubeadm version
> sudo kubelet --version
> sudo kubectl version
> sudo systemctl status kubelet

```
Active: activate (running)
sudo journalctl -u kubelet
```
- status 확인 시 *active(running)* 이 아닐 경우 아래 클러스터 구축의 init문까지 진행 후 다시 status 확인
- 참조 : https://velog.io/@sororiri/k8s-%ED%8A%B8%EB%9F%AC%EB%B8%94%EC%8A%88%ED%8C%85-kubelet-%EC%9D%B4-%EB%8F%99%EC%9E%91%ED%95%98%EC%A7%80-%EC%95%8A%EB%8A%94-%ED%98%84%EC%83%81

 //*(참고)우분투 18.04 버전일 경우 위의 쿠버네티스 설치과정을 master, node1, node2에 각각 해줘야 함*

#### 우분투 20.04 버전 워커노드 생성
- kubelet 까지 진행 후 VM 종료
> exit
> ctrl + ]

- 도커 및 쿠버네티스 까지 설치 후 해당 이미지 복사
> cd /data/kvm/images
> sudo cp -r ./kvm_kube-master.qcow2 ./kvm_kube-node1.qcow2
> sudo cp -r ./kvm_kube-master.qcow2 ./kvm_kube-node2.qcow2

- 워커 노드 실행
>  sudo virt-install --name kube-node1 --disk path=/data/kvm/images/kvm_kube-node1.qcow2 --import --vcpus 2 --ram 4096 --os-type linux --os-variant ubuntu20.04 --network bridge=br0 --graphics none

> sudo virt-install --name kube-node2 --disk path=/data/kvm/images/kvm_kube-node2.qcow2 --import --vcpus 2 --ram 4096 --os-type linux --os-variant ubuntu20.04 --network bridge=br0 --graphics none

- VM 접속 후 각각 ip 변경
> sudo nano /etc/netplan/00-installer-config.yaml

-> addresses: 아래 ip로 입력
  - kube-noe1: [192.168.10.224/24]
  - kube-node2: [192.168.10.225/24]

- 이후 테스트 진행
> sudo netplan apply
> sudo apt update

- 여기까지 진행 한 후 VM hostname 변경 권장 (kube-node1 일때)
> sudo hostnamectl set-hostname kube-node1
> sudo reboot
### 클러스터 구축
- 쿠버네티스 클러스터 : 작동 중인 쿠버네티스 배포를 *클러스터*라고 하며, 클러스터는 리눅스 컨테이너를 실행하는 호스트 그룹 
##### _*kube-master*에서 진행_ 
> sudo rm /etc/containerd/config.toml
> sudo systemctl restart containerd
> sudo kubeadm init --ignore-preflight-errors=all

 - 실행 결과
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  //권한 설정 명령어
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

//아래 코드 복사
kubeadm join 192.168.10.223:6443 --token 5dnoov.jumcqsei2vyfziau \
        --discovery-token-ca-cert-hash sha256:ca8cc20b024733e34992e36199db097c9fc64c08e297fd0fac23b88c92e7923d
```

- 권한 설정
> mkdir -p $HOME/.kube && sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config

- master에서 'sudo kubeadm init --ignore-preflight-errors ...' 명령어 실행 후 나오는 코드 중 맨 마지막 부분 복사
  - > kubeadm join 192.168.10.223:6443 --token 5dnoov.jumcqsei2vyfziau \
        --discovery-token-ca-cert-hash sha256:ca8cc20b024733e34992e36199db097c9fc64c08e297fd0fac23b88c92e7923d

***

##### _*kube-node1, kube-node2*에서 진행_
- 복사된 kubeadm적용 (sudo)
> sudo kubeadm join 192.168.10.223:6443  --token ~~~

- 실행 결과
```
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

- (참고) 노드에서 kubeadm join 실행 시 [ERROR CRI]: container runtime is not running 에러가 발생 시
> sudo nano /etc/containerd/config.toml

- disabled_plugins=["cri"] 부분을 아래와 같이 "cri" 부분만 제거
```
disabled_plugins=[]
```

- 이후 컨테이너 재시작
> sudo systemctl restart containerd

- 다시 kubeadm join 구문 실행
> sudo kubeadm join 192.168.10.223:6443  --token ~~~
***

##### _*kube-master*에서 진행_
- (정상적으로 동작 시 This node has joined the cluster: 라는 메시지가 표시됨)
- node 상태 확인
> kubectl get nodes
- 결과 (kube-node1, kube-node2가 목록에 표시된다면 완료)
```
insight@kube-master:~$ kubectl get nodes

NAME          STATUS     ROLES           AGE    VERSION
kube-master   NotReady   control-plane   3h5m   v1.29.5
kube-node1    NotReady   <none>          18m    v1.29.5
kube-node2    NotReady   <none>          70s    v1.29.5
```

> kubectl get po -n kube-system
```
insight@kube-master:~$ kubectl get po -n kube-system

NAME                                  READY   STATUS    RESTARTS   AGE
coredns-76f75df574-5w99p              0/1     Pending   0          3h5m
coredns-76f75df574-7xvp2              0/1     Pending   0          3h5m
etcd-kube-master                      1/1     Running   0          3h5m
kube-apiserver-kube-master            1/1     Running   0          3h5m
kube-controller-manager-kube-master   1/1     Running   0          3h5m
kube-proxy-hlkw2                      1/1     Running   0          3h5m
kube-proxy-j9sjt                      1/1     Running   0          96s
kube-proxy-kcst7                      1/1     Running   0          18m
kube-scheduler-kube-master            1/1     Running   0          3h5m
```

- flannel 설치
> kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

- 실리움(cilium) 설치
> sudo curl -LO https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
> sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
> rm cilium-linux-amd64.tar.gz
> cilium install
> cilium status
- 실행 결과
```
insight@kube-master:~$ cilium status

    /¯¯\
 /¯¯\__/¯¯\    Cilium:             1 errors, 3 warnings
 \__/¯¯\__/    Operator:           1 errors, 1 warnings
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

Deployment             cilium-operator    Desired: 1, Unavailable: 1/1
DaemonSet              cilium             Desired: 3, Unavailable: 3/3
Containers:            cilium-operator    Pending: 1
                       cilium             Pending: 3
Cluster Pods:          0/2 managed by Cilium
Helm chart version:
Image versions         cilium             quay.io/cilium/cilium:v1.15.4@sha256:b760a4831f5aab71c711f7537a107b751d0d0ce90dd32d8b358df3c5da385426: 3
                       cilium-operator    quay.io/cilium/operator-generic:v1.15.4@sha256:404890a83cca3f28829eb7e54c1564bb6904708cdb7be04ebe69c2b60f164e9a: 1
Errors:                cilium             cilium                              3 pods of DaemonSet cilium are not ready
                       cilium-operator    cilium-operator                     1 pods of Deployment cilium-operator are not ready
Warnings:              cilium             cilium-chkv5                        pod is pending
                       cilium             cilium-t78th                        pod is pending
                       cilium             cilium-tlf75                        pod is pending
                       cilium-operator    cilium-operator-5d64788c99-jwbmb    pod is pending
```

> kubectl get pods --namespace=kube-system -l k8s-app=cilium
- 실행 결과
```
insight@kube-master:~$ kubectl get pods --namespace=kube-system -l k8s-app=cilium

NAME           READY   STATUS     RESTARTS   AGE
cilium-chkv5   0/1     Init:0/6   0          79s
cilium-t78th   0/1     Init:0/6   0          78s
cilium-tlf75   0/1     Init:0/6   0          78s
```

> kubectl get po -n kube-system
- 실행 결과
```
insight@kube-master:~$ kubectl get po -n kube-system

NAME                                  READY   STATUS              RESTARTS      AGE
cilium-chkv5                          1/1     Running             0             2m16s
cilium-operator-5d64788c99-jwbmb      1/1     Running             0             2m14s
cilium-t78th                          1/1     Running             0             2m15s
cilium-tlf75                          1/1     Running             0             2m15s
coredns-76f75df574-5w99p              0/1     ContainerCreating   0             3h9m
coredns-76f75df574-7xvp2              0/1     ContainerCreating   0             3h9m
etcd-kube-master                      1/1     Running             0             3h9m
kube-apiserver-kube-master            1/1     Running             0             3h10m
kube-controller-manager-kube-master   1/1     Running             1 (82s ago)   3h10m
kube-proxy-hlkw2                      1/1     Running             0             3h9m
kube-proxy-j9sjt                      1/1     Running             0             6m2s
kube-proxy-kcst7                      1/1     Running             0             23m
kube-scheduler-kube-master            1/1     Running             1 (82s ago)   3h9m
```

> watch kubectl get po -n kube-system 
- ubuntu 20.04 버전으로 진행 시 coredns AGE가 10분 쯔음 진행되었을 때, ContainerCreating -> Running으로 바뀌면서 ContainerCreating 상태에서 멈추는 문제가 해결됨



***

vm 명칭 error (원인 : 언더바 포함)
```
nodeRegistration.name: Invalid value: "kube-node2_2": a lowercase RFC 1123 subdomain must consist of lower case alphanumeric characters, '-' or '.', and must start and end with an alphanumeric character (e.g. 'example.com', regex used for validation is '[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*')
To see the stack trace of this error execute with --v=5 or higher
```
- 해당 에러 발생 시, 호스트명 변경 후 재부팅, 이후 다시 시도
> sudo hostnamectl set-hostname {변경할 호스트명}
> sudo reboot

