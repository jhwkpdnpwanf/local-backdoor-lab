# local-backdoor-lab

> 이 프로젝트는 오로지 보안 학습 목적입니다.  
> 모든 실습은 로컬 환경(VMware)에서만 실행되며, 실제 외부 시스템을 대상으로 실행하지 않습니다.  
> 학습 용도로만 사용해주세요.
<br>

### 학습 환경  
* **가상화 환경**: VMware Workstation Pro
* **공격 대상 환경**: Kali Linux (VM)
* **호스트 OS**: Windows 11 (WSL 사용)
<br>



### 실습 커리큘럼

1. **리버스 셸 실습**  
   * 리버스 셸 개념 소개

2. **백도어 제작**  
   * crontab, systemd, bashrc, Windows Startup 등 다양한 지속화 기법 실습
   * 프로세스/포트 위장, 로그 제거, 파일 숨기기 등 백도어 은폐 실습

3. **탐지 및 분석 실습**  
   * ps, netstat, lsof, sha256sum, bash\_history, 로그 분석 등을 통한 탐지 실습

4. **커널 백도어 실습**  
   * boopkit과 같은 LKM 기반 백도어 설치, 트리거, 은폐 실습

5. **백도어 탐지기 제작**  
   * python을 이용한 간단한 백도어 탐지기 구현

