# 리눅스(+ K8s) 설치
-  *LINUX OS*(centos, ubuntu, rocky) 가 설치된 usb 준비
-  윈도우 환경 재부팅 후 바이오스 진입(F2 또는 del) > usb로 부팅(우측 목록에서 usb를 제일 상단으로) > F10(save and exit)
- troubleshooting 화면 뜨면 enter 눌러주기
- 한국어 선택 화면 또는 설치 요약 페이지가 뜨면 완료

> 해당 부분은 터미널 실행 명령부분

---
	명령문 실행시 표시되는 터미널 화면

---

```
파일내에 작성해야할 텍스트
```

## Rocky 설치
- centos 대안으로 무료 리눅스 배포판

### 1. 설치 환경 진입
- 설치 목적지 드라이브 설정 (*설치 툴에서 드라이브*)
	- 시스템 - 설치 목적지  
  465.76GiB 선택 > 저장소 구성: '자동 설정' 체크 > 완료 >  
  공간 확보 > 모두 삭제 > 공간 확보
  - 저장 후 시스템 설치 목적지에 '자동 파티션이 선택됨' 이라고 뜨면 완료
  
- root 비밀번호 설정 120607q!@# (*ssh 접속 허용 체크*)
  - 'root가 비밀번호로 ssh에 로그인하도록 허용' 체크
  - (ssh 접속 허용 시 타PC 터미널로 접근 가능)

- 사용자 추가
	- insight / 120607a!@#
  - '이 사용자를 관리자로 설정합니다.' 체크
  - (리눅스 로그인 시 해당 계정으로 진입)

- 네트워크 설정(*툴에서 네트워크 설정*)
  - (enp2s0으로 설정되어 있는 건 그대로 두고 추가로 설정)
  - 네트워크 및 OO 설정 > 우측 하단 설정 > ipv4설정 > 주소, 넷마스크, 게이트웨이, DNS servers에 아래 정보 각각 입력
	- 주소: 192.168.10.222 
	- 넷마스크 : 255.255.255.0 
	- 게이트웨이 : 192.168.10.1
	- DNS servers : 1.214.68.2, 61.41.153.2  

### 2. 리눅스 진입
- 부팅 후 등록한 사용자로 로그인
- 1번에서 설정한 ip 확인
> ip ad
> ifconfig

### 3. 디스크 추가 (700G 드라이브 마운트)

-  root 유저로 진행
> su -
> fdisk -l
> fdisk /dev/sdb1
- 이후 *n p 1 enter enter w* 순서대로 입력 (**테스트 시 w는 적용되지 않았음**)
- n : 새 파티션 생성
- partition type : p (Primary 파티션으로 선택)
- partition number : 1 (파티션 번호)
- first sector : enter
- last sector : enter
- w : 파티션 정보 저장

> mkfs.xfs -i size=512 -f /dev/sdb1  
> mount /dev/sdb1 /data (띄어쓰기 주의)  
> df -h  

- 재부팅 시 자동 mount 설정
> su -  
> echo '/dev/sdb1 /data xfs defaults 1 2' >> /etc/fstab

- HDD 구조 확인 (**현재 드라이브 500G, 추가된 700G 드라이브 목록에 나타나는지 확인**)
 > lsblk
 
 - sudoers 유저 추가
> usermod -aG wheel insight

- 업데이트
> dnf update
 - ...진행할까요? 질문이 두 번 뜨는데 둘 다 y 눌러주기
 - (오류) repo를 위한 메타자료 내려받기에 실패하였습니다  
  : ip설정(네트워크 설정)이 안돼서 발생한 오류

### 4. 보안 설정
- root 로 진행
> su -
> nano /etc/ssh/sshd_config
 - 아래 내용 입력 후 ctrl + x
 - 수정한 버퍼 내용을 저장하시겠습니까? y
 - 기록할 파일이름 : 기존 파일이름 그대로 enter

```
Port 11800
ClientAliveInterval 100
ClientAliveCountMax 3
MaxAuthTries 3
```
- Port : ssh 접속 포트번호 설정
- ClientAliveInterval : 클라이언트 살아있는지 확인하는 간격
- ClientAliveCountMax : 클라이언트 응답이 없어도 접속 유지하는 횟수, (*해당 2개 설정으로 100 * 3초 (5분) 후 접속이 끊어짐*)
- MaxAuthTries : 로그인 시도횟수 제한

- ssh root 접속 불가 설정 (보안처리)
> nano /etc/ssh/sshd_config.d/01-permitrootlogin.conf
```
PermitRootLogin no
```
- 방화벽 내 11800 포트 허용

> firewall-cmd --permanent --zone=public --add-port=11800/tcp  
> semanage port -a -t ssh_port_t -p tcp 11800  
> systemctl restart sshd  
> netstat -tnl  
> reboot  

- 다른 PC에서 접속 확인
> ssh insight@192.168.10.222 -p 11800  
> id/pw : insight / 120607a!@#

- 접속 시 아래와 같이 오류 발생 시,  
C:\Users\insight/.ssh 경로로 가서 known_hosts 파일 내 192.168.10.222:11800의 rsa 키 삭제해주기
```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ECDSA key sent by the remote host is
SHA256:Dhz57Cd0WlIdPJaPuihSNmR4LH1kO/iI1a2Oiv4yOwM.
Please contact your system administrator.
Add correct host key in C:\\Users\\insight/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in C:\\Users\\insight/.ssh/known_hosts:6
ECDSA host key for [192.168.10.222]:11800 has changed and you have requested strict checking.
```
- 이후 재접속하면 아래와 같은 메시지가 뜨는데 yes 입력 후 진행
```
The authenticity of host '[192.168.10.222]:11800 ([192.168.10.222]:11800)' can't be established.  
ECDSA key fingerprint is SHA256:b32/0GP2i8Lh4zAzu5L55bTaAWWpIMU2Yc7DgnFFbBE.  
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

### 5. IP 설정 (1번에서 네트워크 설정 안했을 시)
- 수동입력 방식으로 진행함
> nmcli con mod enp2s0 ipv4.address '192.168.10.222'  
> nmcli con mod enp2s0 ipv4.gateway '192.168.10.1'  
> nmcli con mod enp2s0 ipv4.method manual  
> nmcli con mod enp2s0 ipv4.dns '1.214.68.2,61.41.153.2'  
> nmcli con down enp2s0  
> nmcli con up enp2s0  

- 설정 후 인터넷 연결 테스트 및 네트워크 연결 확인
> ping -c3 google.com  
> nmcli dev status  
> nmcli con show  
> ip a  
- 위 방법으로 네트워크 설정 시 /etc/sysconfig/network-scripts 에 정상적으로 파일이 만들어지지 않았음

-> *위 항목 까지의 과정은 터미널 ssh로 타PC에서 리눅스 본장비에  접속하기 위한 필수 작업임*

