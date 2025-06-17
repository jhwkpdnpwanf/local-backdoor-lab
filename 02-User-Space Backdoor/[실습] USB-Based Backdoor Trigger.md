# USB 이벤트 감지 트리거 백도어

기본적인 시간 기반이나 사용자 기반(`.zshrc`) 외에 더 은밀한 방식을 체험해보고 인식하기 위한 실습이다.  
시스템 내부 이벤트, 특히 물리적인 하드웨어 동작을 활용한 자동 실행 방식이 존재한다.  

usb 포트와 같은 하드웨어 동작과 관련된 공격 방식은 매우 다양하고 위험하기 때문에 실제 보안 환경에서 usb를 완전 비활성화 하거나 봉인해두는 경우가 많다.  
<br>
<br>

### 백도어 구현
- **상황** : 리눅스에서 루트 계정에 일시적인 쉘 접근 권한을 가진 상태.
- **목표** : USB가 연결되면 USB 내부 파일을 읽고 공격자 컴퓨터에 전송하도록 만들기.
- **방법** : `udev` 장치 이벤트를 감지해서 전송하기.
<br>

USB가 연결되면 usb 내부 파일을 읽고 공격자 컴퓨터에 전송해주는 코드를 만들어 볼 것이다.  
<br>

**0. VM 의 USB 설정을 바꿔준다.**

가상환경에서 실습을 하고 있으므로 kali VM 의 USB 설정을 바꿔줘야한다.     
USB 연결을 VM 내부로 연결되도록 아래 설정을 눌러준다. 아래 화면은 usb 연결이 감지되면 화면에 뜬다.  

<br>

<img src="https://github.com/user-attachments/assets/ac36c7e3-28fb-46e7-80ca-ee4648cdc3d6" width=500>  


Connect to a virtual machine 을 누르고 Debian 12.x 를 선택해주면 잘 연결된다.   
<br>

**1. USB 연결 감지 확인**    

usb가 잘 연결되는지 확인하는 단계이다.   
<br>

<img src="https://github.com/user-attachments/assets/cab8efbe-4584-4ddd-9791-c6f89b34912d" width=800>  

`lsusb`로 usb 연결 상태를 볼 수 있다.  

`Bus 001 Device 003: ID 058f:6387 Alcor Micro Corp. Flash Drive`   

이렇게 1번 버스에 usb가 연결되어 있는걸 확인할 수 있다.  
<br>

**2. udev 구조 파악**  

udev는 커널의 장치 이벤트를 감지하고,   
이벤트 조건에 따라 지정된 스크립트를 자동 실행할 수 있다.  

단 `udev`는 커널 권한이 필요하므로 root 쉘을 얻었다 가정하고 스크립트를 작성해주겠다.   

udev 규칙 적용 파일 위치
- /etc/udev/rules.d/ → 커스텀 규칙
- /run/udev/rules.d/ → 런타임 임시 규칙
- /lib/udev/rules.d/ → 시스템 기본 규칙



<img src="https://github.com/user-attachments/assets/70076474-dc6b-440f-9f19-61b27eacdac6" width=800>   

이런식으로 저장되어있다. (시스템 기본 규칙)   
참고로 숫자가 작을수록 먼저 적용이된다.   
<br>

커스텀한 규칙을 저장할 것이므로,  
udev 설정 파일은 `/etc/udev/rules.d/` 위치에 저장하면 된다.   

<img src="https://github.com/user-attachments/assets/8a0b0c28-683c-44bd-80c5-8fb6edf628bb" width=500>   

해당 경로에 `/etc/udev/rules.d/99-usb-backdoor.rules` 파일을 만들 것이다.  
<br>

**3. udev 규칙 작성.**  

규칙 내용으로는 usb 연결 시 백도어 스크립트 `.sh` 파일을 실행하게 하자.  

백도어 스크립트는 /usr/local/bin/usb_backdoor.sh 에 둘것이다.   

```bash
sudo nano /etc/udev/rules.d/99-usb-backdoor.rules

ACTION=="add", SUBSYSTEMS=="usb", ENV{DEVTYPE}=="usb_device", RUN+="/usr/local/bin/usb_backdoor.sh"
```

USB 장치 연결 시에 뭐가 연결되면 일단 /usr/local/bin/usb_backdoor.sh 를 실행한다.   




![image](https://github.com/user-attachments/assets/e1bd7f23-31d9-4475-8810-cd63e4b40d2a)


[해결해야할것들]  
usb 인식 할 시간 줘보기 
udev 지속실행 (너무 빨리 실행) 
스크립트는 실행되는데 마운트가안됨   
