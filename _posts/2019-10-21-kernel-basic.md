---
title: Kernel Basic
categories: Kernel
tags: [theory]
---

## 목차

- 커널 보호기법
- 리눅스 커널 모듈 - 명령어
- 리눅스 커널 모듈 - 함수
- user mode -> kernel mode
- kernel mode -> user mode
- 기본 설정



<hr/>
## 커널 보호기법

###`KASLR` (Kernel Address Space Layout Randomization)

  - **커널의 메모리 주소를 랜덤화** 시킨다.

###`SMEP` (Supervisor Mode Execution Protection)

  - **유저 영역의 주소 공간에서 커널의 코드를 실행을 방지**한다.

###`SMAP` (Supervisor Mode Access Protection)

  - **유저 영역의 주소 공간에서 메모리 접근을 비허용**한다.

###`KADR` (Kernel Address Display Restriction)

  - **비 루트 사용자가 커널 주소의 정보를 얻는 것을 제한**한다.

  - 모든 커널 심볼이 저장된 `/proc/kallsyms`파일의 심볼 주소가 0으로 표시된다.

###`KPRI` (Kernel Page Table Isolation)

  - **유저 공간에서 커널 공간의 데이터 획득을 막기 위해 page를 따로 사용하는 것**

###`canary`

  - **모듈에 카나리가 있을 경우 BOF를 방지**한다.



<hr/>
## 리눅스 커널 모듈 - 명령어

###`lsmod`

  - **실행중인 모듈 리스트**를 확인할 때 사용

###`insmod` [모듈 명]

  - **커널에 모듈 삽입**할 때 사용

###`modinfo` [모듈 명]

  - **모듈 정보**를 볼 때 사용

###`rmmood` [모듈 명]

  - **모듈을 삭제**할 때 사용



<hr/>
## 리눅스 커널 모듈 - 함수

###`module_init`

  - **모듈을 적재할 때 실행**하는 함수

  - 성공하면 0을 반환

###`module_exit`

  - **모듈을 삭제할 때 실행**되는 함수

###`printk`

  - `printf`와 비슷한 함수

  - printf는 `stdout`에 출력하지만, printk는 `kernel ring buffer`에 값을 출력

  - `dmesg` 명령이나 `/var/log/messages` 또는 `/var/log/kern.log` 파일에서 확인 가능

###`kmalloc`

  - `malloc`과 비슷한 함수

  - kmalloc는 `ptmalloc2`가 아닌 `slab allocator`를 통해 메모리를 할당

###`kfree`

  - `free`와 비슷한 함수

  - kfree는 `ptmalloc2`가 아닌 `slab allocator`를 통해 메모리를 해제

###`copy_from_user`

  - `memcpy`와 비슷한 함수

  - **유저 영역에서 커널 영역으로 특정 바이트 만큼 복사**

  - **정상적으로 수행했으면 0을 반환**한다. (아니라면 복사되지 않은 바이트 수)

###`copy_to_user`

  - `memcpy`와 비슷한 함수

  - **커널 영역에서 유저 영역으로 특정 바이트 만큼 복사**

  - **정상적으로 수행했으면 0을 반환**한다. (아니라면 복사되지 않은 바이트 수)

###`prepare_kernel_cred`

  - **새로운 cred 구조체를 생성**하는 함수

  - **인자를 0으로 주면 내부 동작에 의해 uid, gid가 0으로 설정된 cred가 생성** 된다. (root 권한)

  > #### cred 구조체
  >
  > 커널은 현재 프로세스의 권한 정보를 cred라는 구조체에 저장하는데, uid, gid 등 여러가지 정보를 담고 있음

###`commit_creds`

  -> 인자로 cred 구조체를 넘겨주면 새로운 권한을 적용한다.

  

<hr/>
## user mode -> kernel mode

1. **`swapgs` 인스트럭션을 통해 `GS` 레지스터와 `MSR` 레지스터의 `KERNEL_GS_BASE` 값을 교환**한다.
2. **user mode의 레지스터들을 백업하기 위해 스택에 push**한다.
3. 커널 전역변수 `sys_call_table`에 접근해 syscall을 처리한다.



<hr/>
## kernel mode -> user mode

1. `swapgs`로 `GS` 레지스터를 복구한다.
2. `sysretq` / `iretq`로 유저 영역에 복귀한다.



<hr/>
## 기본 설정

기본적으로 CTF Kernel 문제에서 제공되는 파일은 다음과 같다. (보통 bzImage, cpio, sh 파일이 주어짐)

###`bzImage`

  - **압축된 커널 이미지로 부팅시 사용**한다.

  - **해당 파일을 이용해서 vmlinux 파일을 추출** 가능하다.

###`cpio`

  - **file system을 압축시켜놓은 파일**이다.

  - `file` 명령을 이용해서 압축 파일인 지 확인하고 **압축 파일이라면 먼저 해제**하면 된다.

  - **cpio 파일에서 ko 파일을 얻을 수 있다.**

###`sh`

  - **qemu의 실행 옵션이 들어있는 셸 스크립트 파일**이다.

###`ko`

  - **모듈 파일로서 IDA를 이용해서 분석**해야 하는 파일이다.

###`vmlinux`

  - **커널 컴파일 시 생성되는 이미지 파일**이다.




#### vmlinux 파일 추출하기

- **설정**
	
  ```bash
  sudo apt-get install linux-headers-$(uname -r)
  sudo ln -s /usr/src/linux-headers-$(uname -r)/scripts/extract-vmlinux /usr/bin/extract-vmlinux
  ```

- **추출**

  ```bash
  extract-vmlinux bzImage > vmlinux
  ```




#### cpio 압축 해제

- 보통 `gzip`으로 압축되어 있는 경우가 많은데, 이땐 다음과 같은 명령으로 해제하면 된다.

  ```bash
  mv *.cpio *.cpio.gz
  gzip -d *.cpio.gz
  ```


- 그 이후 다음 명령어를 이용해 압축을 해제한다.

  ```bash
  cpio -idm < *.cpio
  ```


  ​	- i: 압축 해제

  ​	- d: 없는 디렉토리 생성

  ​	- m: 변경시간 유지

- **ko 파일은 압축해제해서 나온 file system의 루트 디렉토리 또는 lib 디렉토리 내부에 존재**한다.




#### sh 파일 수정

- -m 옵션이 64M일 경우 256M으로 변경 (64는 부족함)

- -s 옵션이 없을 경우 추가

  - **1234번 포트로 연결을 기다리는 옵션**

  - p 옵션으로 원하는 포트 지정도 가능




#### 부팅

- `qemu`가 설치 안되었다면 설치한다.

	```bash
	sudo apt-get install qemu
	```

- sh 파일에 `enable kvm`이 있다면 다음 설정을 한다.

  ```
  vmware에서 Settings -> Processors -> Virtualize Intel VT-x/EPT or AMD-V/RVI 체크 (가상 머신이 종료된 상황에서)
  ```

- 설정이 끝났다면 sh파일을 실행한다.




<hr/>
## 레퍼런스

- https://st4nw.github.io/2019-09-02/kadr/
- https://sunrinjuntae.tistory.com/124?category=732710
- https://sunrinjuntae.tistory.com/125?category=732710
- https://st4nw.github.io/2019-09-08/kernel3/
- https://st4nw.github.io/2019-09-08/kernel2/
- https://applemasterz17.tistory.com/230
- https://defenit.kr/2019/10/18/Pwn/ㄴ WriteUps/CISCN-2017-babydriver-Write-Up-linux-kernel-UAF/
- https://rajalo.tistory.com/entry/cpio-옵션
- https://m.blog.naver.com/PostView.nhn?blogId=kjug1589&logNo=221598098507&categoryNo=0&proxyReferer=https%3A%2F%2Fwww.google.co.kr%2F

