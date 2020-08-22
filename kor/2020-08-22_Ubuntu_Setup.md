# 우분투 서버 세팅
최초 작성일: Aug. 22. 2020


#### Table of Contents
- Introduction
- Root Password Change and New Admin User
- SSH Setup (Port TCP 22)
- 로케일 설정
- apt 저장소 갱신 및 패키지 업데이트
- 자주 사용하는 패키지 목록
- Conclusion


### Introduction
우리가 컴퓨터를 새로 사면 컴퓨터를 사용하기 전 비밀번호를 설정하고 프로그램을 깔기 시작하듯, 서버도 새로 구매한 뒤 기본적인 설정 절차가 필요하다.
컴퓨터도 개인마다 세팅하는 방법이 다르듯이, 서버도 사용자의 취향에 따라 세팅이 달라질 수 있다.  

이 포스트는 우분투 20.04 LTS 환경에서의 세팅 방법을 다루지만, 데비안 기반의 다른 리눅스 배포판에서도 비슷한 명령어들을 통해 세팅을 진행할 수 있으리라 생각된다.


### Root Password Change & New Admin User
기본적으로 호스팅 업체에서는 서버에 접근 가능한 root 계정 정보를 전달해 줄 것이다.
기본 비밀번호로 계속 사용해도 무관하지만, 만일에 대비해 root 게정의 비밀번호를 미리 바꾸어 두는 것을 추천한다.  

아래 명령어는 제공받은 root 게정으로 서버에 접근한 상태에서 실행한다.
루트 상태에서는 본인의 게정정보를 포함한 다른 게정의 정보도 바꿀 수 있다.

**root 계정 패스워드 변경**  
root로 로그인한 상태에서 `passwd` 명령어를 통해 현재 계정(root)의 비밀번호를 재설정할 수 있다.

**새 관리자 계정 생성**  
root 계정은 시스템 전체의 권한을 쥐고있다.
root 계정을 직접 접근하여 사용하여 항상 root 권한을 쥐고 있는것보다는, 새 관리자 게정을 생성하고 이 게정이 필요할 때만 `sudo` 명령을 통해 root 권한을 사용하여 작업할 수 있도록 설정해 두는 것이 더 바람직하다.

1. 새 유저 생성
   ```
   useradd -m -s /bin/bash [account_name]
   ```
   `[account_name]` 부분에 새로 생성하고 싶은 계정의 이름을 적은 상태로 위 명령어를 실행한다.
   위 명령은 계정의 홈 디렉터리와와 쉘 환경(bash shell)을 동시에 지정한다.

[sudoers 등록]  

새 계정이 `sudo` 명령을 사용할 수 있도록 `sudoers ` 파일에 게정을 등록한다.   

계정 등록방법에는 두가지가 있다.
이미 `sudo` 권한이 부여된 그룹에 계정을 등록하는 방법이 있고, 계정을 직접 `sudoers` 파일에 등록하는 방법이 있다.
나는 계정을 일일히 `sudoers` 파일에 등록하는 것 보다는 `sudo` 권한이 부여된 그룹에 계정을 등록하는 편이 관리의 편의성이 높다고 생각하여, 이 방법으로 새로 만든 게정에 `sudo` 권한을 부여하겠다.

먼저, 어떤 그룹에 `sudo` 권한이 부여되어 있는지를 확인한다.
```
cat /etc/sudoers | grep %
```
실행 결과에서 `%` 기호 뒤에 나오는 이름이 `sudo` 권한이 부여된 그룹의 이름이다.
여러개가 나오면 이 중 하나의 그룹에만 계정이 포함되어 있어도 그 게정으로 `sudo` 명령어를 사용할 수 있다.

다음으로, 그룹이 존재하는지를 확인한다.
```
cat /etc/group | grep "[group_name]"
```
`[group_name]` 에 찾고자 하는 그룹 이름을 넣는다.  

만약 그룹이 존재하지 않는다면, 새로 그룹을 만들어 주어야 한다.
```
groupadd [group_name]
```
`[group_name]` 에 만들고자 하는 그룹 이름을 넣는다.  

그룹이 존재하거나, 새로 그룹을 만든 이후에 계정을 그룹에 등록한다.
```
gpasswd -a [account_name] [group_name]
```
`[account_name]`에 추가 대상 계정 이름을, `[group_name]` 에 앞에서 정의한 게정을 추가하고자 하는 그룹 이름을 넣는다.  

[계정 비밀번호 생성]  
```
passwd [account_name]
```
새로 만든 계정에 `sudo` 명령을 사용할 수 있는 권한은 주었지만, 아직 계정에 비밀번호는 설정되지 않은 상태이다.  
계정의 비밀번호를 세팅해주도록 하자.


### SSH Setup (Port TCP 22)
SSH는 Secure Shell의 약어로, 원격 호스트에 접속하기 위해 사용하는 인터넷 프로토콜을 지칭한다.  
보통, 가상 서버를 새로 세팅하게 되면, SSH 호스트가 기본적으로 설치되어 있다.  

보안을 위해, `root`계정으로의 직접 접속차단, ChallengeResponseAuthentication 비활성화, 편의를 위한 비밀번호 로그인 허용, 그리고 SFTP 사용 허용으로 설정한다.  

SSH 서버의 설정 파일은 `/etc/ssh/sshd_config`이며, 이를 `nano` 혹은 `vim` 등의 텍스트 에디터를 통해 열어준다.
파일의 소유자가 `root`이므로, 이 파일을 열기 위해서는 루트권한이 필요하다.
현재 로그인된 사용자가 `root`가 아닌 경우 `sudo`를 명령어 앞에 붙여야 한다.
```
sudo nano /etc/ssh/sshd_config
```

주석(`#`로 시작)상태로 설정이 존재하는 경우, 주석을 해제하여야 변경된 설정이 반영된다.

**PermitRootLogin**: `root` 사용자의 로그인 허용 여부를 나타낸다.
우리는 루트 사용자의 ssh 직접 접속을 차단할 것이다.  
**`no`로 설정**  

**PasswordAuthentication**: 비밀번호를 사용한 로그인의 허용 여부를 나타낸다.  
**`yes`로 설정**

**ChallengeResponseAuthentication**: Challenge-response 암호를 활용한 인증 허용 여부를 나타낸다.
우리는 이 기능을 활용하지 않을 예정이다.  
**`no`로 설정**  

**SFTP 사용**: SSH 프로토콜을 활용해 파일을 주고받을수 있는 기능이다.
별도로 FTP 서버를 구축하지 않아도 SFTP를 활용해 파일을 주고받을수 있으므로, 관리의 편의를 위해 SFTP를 사용하도록 세팅한다.  
이미 관련 `subsystem`이 `openssh`에 포함되어 있어 서브시스템 모듈을 불러오기만 하면 된다.  
**`Subsystem       sftp    /usr/lib/openssh/sftp-server` 주석 해제** 혹은  
**`Subsystem       sftp    /usr/lib/openssh/sftp-server` 추가**  

설정 파일을 수정한 이후, 설정 파일을 적용해 주어야 한다.
```
sudo service sshd reload
```

만일에 대비해 **현재 연결을 끊지 말고**, 새로 SSH 연결을 시도해본다.  
`root` 계정으로 로그인을 시도해보고 (연결이 되지 않아야 정상), 새로 추가한 게정으로 정상적으로 로그인이 되는지 확인한다.


### 로케일 설정
타임존(시간대) 및 시스템 언어 등을 설정한다.

[타임존 설정]  

사용할 수 있는 타임존 목록에서 사용할 타임존을 찾는다.
```
timedatectl list-timezones
```

아래 명령어를 실행해 타임존을 설정한다.
루트 권한을 필요로 한다.  
```
sudo timedatectl set-timezone '[time_zone]
```
`[time_zone]`에 사용할 타임존의 이름을 적어준다.

[시스템 언어 설정]  

한글 로케일(`ko_KR.utf8`)의 경우 대부분의 해외 업체 서버를 사용하는 경우 기본적으로 설치되어 있지 않다.
이 경우 한글 파일 이름 및 한글 사용에 문제가 생길 가능성이 있으므로, 로케일을 추가해준다.  

기본 시스템 언어는 그대로 놔두면서 한글 로케일만 추가한다. 루트 권한이 필요하다
```
sudo locale-gen ko_KR.utf8
sudo update-locale
```

로케일 설정 추가 확인
```
locale -a
```


### apt 저장소 갱신 및 패키지 업데이트
apt 저장소는 데미안 계열 리눅스에서 사용하는 기본 패키지 저장소이다.
서버에 기본 설치된 패키지들을 최신 버전으로 유지하기 위해 apt 저장소를 주기적으로 갱신하고 패키지를 업데이트 해 주어야 한다.  

저장소를 갱신하고 패키지를 업데이트 하기 위해서는 루트 권한이 필요하다.
```
sudo apt update
sudo apt upgrade
```

위 두개의 명령어는 각각 저장소 갱신 그리고 패키지 업데이트를 위한 명령어이다.


### 자주 사용하는 패키지 목록
아래 리스트는 글쓴이가 자주 사용하는 패키지 목록이다.
일일히 패키지의 설명 아래에 적힌 명령어를 실행하여도 되지만, `apt install` 명령어 뒤에 설치할 패키지의 이름들을 나열하여 동시에 여러개의 패키지를 설치할 수 있다.

#### vim
유닉스 환경에서 주로 활용되는 vi 편집기의 기능을 개선한 텍스트 에디터이다.
```
sudo apt install vim
```

#### build-essential, software-properties-common
소스코드 빌드 및 패키지 설치 시 필요한(자주 사용되는) 각종 패키지들을 한번에 설치한다.
```
sudo apt install build-essential software-properties-common
```

#### AdoptOpenJDK
오라클 라이센스 정책 변경으로 인해 더이상 Oracle JDK를 무료료 사용하지 못한다.
이에 따라, Java SE의 오픈소스 구현체인 OpenJDK를 사용해야 하며, OpenJDK는 다양한 회사/커뮤니티에서 배포한다.  

여기서는 IBM, Microsoft Azure, Red Hat 등이 참여하는 AdoptOpenJDK의 설치에 대해 다룬다.  
Reference: https://adoptopenjdk.net/installation.html#linux-pkg

먼저, AdoptOpenJDK의 공식 GPG키를 받아온다.
```
wget -qO - https://adoptopenjdk.jfrog.io/adoptopenjdk/api/gpg/key/public | sudo apt-key add -
```

이후, AdoptOpenJDK의 deb 리포지토리를 추가한다.
```
sudo add-apt-repository --yes https://adoptopenjdk.jfrog.io/adoptopenjdk/deb/
sudo apt update
```

마지막으로, AdoptOpenJDK 패키지를 설치한다. 이때, 설치하고자 하는 JDK의 버전과 VM 종류(HotSpot 혹은 OpenJ9)를 명시하여야 한다.  
아래는 AdoptOpenJDK 11 (LTS)를 OpenJ9 VM과 함께 설치하는 명령어이다.
```
sudo apt install adoptopenjdk-11-openj9
```
OpenJ9 VM을 설치하고 싶은 경우 버전 뒤에 `openj9`를, HotSpot VM을 설치하고 싶은 경우 버전 뒤에 `hotspot`을 적는다.


#### Python3
파이썬 3 실행환경과 pip를 설치한다.
```
sudo apt install python3 python3-pip
```


### sysstat
cpu, memory, disk IO 등의 사용 정보를 모니터링하기 위한 툴이다.
```
sudo apt install sysstat
```

`sysstat`을 사용하도록 설정해야 한다.  
`/etc/default/sysstat` 파일에서 `ENABLED="false"`를 `ENABLED="true"` 로 변경한다.

10분에 한번 성능 지표가 기록된다.
`crontab`을 통해 성능지표를 매 10분마다 기록하도록 설정되어 있다.  
`crontab` 설정파일 위치: `/etc/cron.d/stsstat`  

설정 완료 후, 서비스를 재시작한다.
```
sudo service sysstat restart
```

구체적인 사용법에 대해서는 추후에 다른 포스트로 정리할 예정이다.


#### Apache (Port 80, TCP 443)
Apache는 세상에서 가장 많이 활용되는 웹서버 중 하나이다.
웹페이지를 호스팅하기 위해 아파치 웹 서버를 설치한다.
```
sudo apt install apache2
```

서버 IP를 웹 브라우저에 입력하면 아파지가 설치되어 실행되고 있다는 메세지를 볼 수 있을 것이다.
실제 사용 방법에 관해선 추후에 다른 포스트로 정리할 예정이다.


#### ufw
우분투의 기본 방화벽이다.
기존의 iptables를 쉽게 설정할 수 있도록 한 것이다.
```
sudo apt install ufw
sudo ufw enable
```

기본 설정 (HTTP(80), HTTPS(443), SSH 허용, 나머지 비허용)
```
sudo ufw default deny
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 443
```

설정 내역 확인
```
sudo ufw status
```

구체적인 사용 방법에 관해서는 추후에 다른 포스트로 정리할 예정이다.


### Conclusion
지금까지 기본적인 리눅스 서버 세팅 방법에 대해 알아보았다.
앞서 언급하였듯이, 사람마다 필요한 기능과 서버 사용의 목적이 다르므로 꼭 이 포스트의 설정 방법이 정답이라고 할 수는 없다.