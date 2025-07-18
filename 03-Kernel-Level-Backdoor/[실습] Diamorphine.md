# Diamorphine 실습
Diamorphine 은 LKM 루트킷 중 하나로, 프로세스 은닉, 은폐, 권한 상승 기능을 제공한다.  
시스템콜을 후킹하여 몇가지 커널 기능을 가로채고 작동하는 방식을 살펴본다.  

<br>

**실습환경**  
- **버전** : Kali Linux 2019.4
- **목표** : rootkit 동작 방식 분석
<br>

자세한 내용을 보고 싶으면 아래 링크에서 볼 수 있다.  
https://github.com/m0nad/Diamorphine

우선 설치해서 실행을 시켜본 뒤 기능들 원리를 살표보자.  
<br>

**0. 우선 깃 클론을 해준다.**  

<img src="https://github.com/user-attachments/assets/ed0599cc-afa1-48ec-aa6d-13130ffcea11" width=800>   

<br>

```bash
git clone https://github.com/m0nad/Diamorphine.git
```


<img width="1054" height="329" alt="image" src="https://github.com/user-attachments/assets/12f6719b-490a-4bcd-bc36-119b1b0bf34a" />

<img width="956" height="312" alt="image" src="https://github.com/user-attachments/assets/062f2145-e457-4863-9e64-2a1b70582e47" />


<img width="802" height="100" alt="image" src="https://github.com/user-attachments/assets/046273e0-ca1a-40ee-a373-f978846dca3d" />


