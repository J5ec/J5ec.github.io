---
layout: post
title: JS Engine Basic
categories: Browser
tags: [theory]
---

## 목차
- JS Engine
- V8
- SPiderMonkey
- Chakra Core
- JavaScript Core
- JS Engine 동작 과정
- 인터프리터 / 컴파일러 동작 과정
  - V8 동작 과정
  - SpiderMonkey 동작 과정
  - Chakra Core 동작 과정
  - JavaScript Core 동작 과정
- JS Array
  - Array 특징
  - Array 함수
- 레퍼런스
- 문서 역사

---

## JS Engine
- 자바스크립트 코드를 실행하는 프로그램 또는 인터프리터
- 대표적으로 `V8`, `SpiderMonkey`, `Chakra Core`, `JavaScript Core`가 있다.

---

## V8 
- C++로 작성되었으며, 구글이 개발한 오픈소스
- `Google Chrome`에서 사용한다.
- ![image](https://user-images.githubusercontent.com/43925259/66770973-e2bcd300-eef3-11e9-915c-a67e99e90863.png)

---

## SpiderMonkey

- 최초의 자바스크립트 엔진으로, JS의 창시자가 브라우저를 위해 개발
- `FireFox`에서 사용
- ![image](https://user-images.githubusercontent.com/43925259/66771031-11d34480-eef4-11e9-96fc-491076db5add.png)

---

## Chakra Core

- 마이크로소프트가 개발한 엔진
- `Edge` 브라우저에서 사용
- ![image](https://user-images.githubusercontent.com/43925259/66771181-7a222600-eef4-11e9-9156-d4693f3d3fa6.png)

---

## JavaScript Core

- 애플에서 WebKit 프레임워크를 위해 개발한 엔진
- `Safari`와 React Native App에서 사용
- ![image](https://user-images.githubusercontent.com/43925259/66771039-17c92580-eef4-11e9-8544-9ee9cebeb449.png)

---

## JS Engine 동작 과정

![image](https://user-images.githubusercontent.com/43925259/66770764-5ad6c900-eef3-11e9-9323-3cb60652df38.png)

1. 자바스크립트 소스 작성
2. 파싱
3. [AST](https://ko.wikipedia.org/wiki/추상_구문_트리) 생성
4. 인터프리터가 AST를 가지고 최적화 되지 않은 바이트 코드를 빠르게 생성
5. 최적화 컴파일러에서 약간의 시간을 들여서 최적화된 기계어 코드 생성
6. 만약 정확하지 않은 결과가 나왔다면 다시 바이트 코드로 변경 (deoptimize)

---

## 인터프리터 / 컴파일러 동작과정

![image](https://user-images.githubusercontent.com/43925259/66771227-98882180-eef4-11e9-9769-8b04257ab1c5.png)
- 자바스크립트 엔진마다 동작 과정에 차이가 있지만 위 과정은 공통이다.

### V8 동작 과정

![image](https://user-images.githubusercontent.com/43925259/66771378-0df3f200-eef5-11e9-9096-c0910b243a8e.png)

- 그림 자체는 공통 그림과 별 차이가 없는데, 네이밍을 자동차 엔진처럼 해놨다.
- `인터프린터`는 `Ignition`이라고 부르며, `최적화 컴파일러`는 `TurboFan`이라고 부른다.

- **Ignition의 역할**
  - 코드를 점화하여 바이트 코드를 생성 및 실행
  - 프로파일링 데이터를 수집
  - 특정 함수가 자주 호출되서 뜨거워졌다면 바이트 코드 및 프로파일링 데이터를 `TurboFan`으로 전송
- **TurboFan의 역할**
  - 프로파일링 된 데이터를 기반으로 매우 최적화된 기계어 코드를 생성 (뜨거워진 함수를 식혀줌)
  - 최적화가 실패하면 다시 바이트 코드로 변경

### SpiderMonkey 동작 과정

![image](https://user-images.githubusercontent.com/43925259/66771393-1a784a80-eef5-11e9-8392-a0441b6b4477.png)
- 스파이더 몽키는 최적화 컴파일러가 2개가 존재한다.

- **인터프리터 역할**
  - 최적화 되지 않은 바이트 코드 생성
- **Baseline 역할** 
  - 약간 최적화된 코드 생성
  - 프로파일링 데이터 수집
- **lonMonkey 역할**
  - 고도로 최적화된 코드 생성
  - 최적화에 실패하면 Baseline 코드로 변경

### Chakra Core 동작 과정

![image](https://user-images.githubusercontent.com/43925259/66771404-2532df80-eef5-11e9-827e-e799e26e9779.png)
- 차크라 코어도 2개의 최적화 컴파일러가 존재한다.
- 스파이더 몽키와 유사한 과정이다.

- **인터프리터 역할**
  - 최적화 되지 않은 바이트 코드 생성
- **Simple JIT 역할** 
  - 약간 최적화된 코드 생성
- **FullJIT 역할**
  - 고도로 최적화된 코드 생성
  - 최적화에 실패하면 바이트 코드로 변경

### JavaScript Core 동작 과정

![image](https://user-images.githubusercontent.com/43925259/66771440-34199200-eef5-11e9-81da-967df7ee7616.png)
- 자바스크립트 코어는 세 가지의 최적화 컴파일러를 사용한다.

- **LLInt 역할** (Low-Level Interpreter)
  - 최적화 되지 않은 바이트 코드 생성
- **Baseline 역할** 
  - 약간 최적화된 코드 생성
- **DFG 역할** (Data Flow Graph)
  - 조금 더 최적화된 코드 생성
  - 최적화에 실패하면 약간 최적화된 코드로 변경
- **FTL 역할** (Faster Than Light)
  - 고도로 최적화된 코드 생성
  - 최적화에 실패하면 약간 최적화된 코드로 변경

---

## JS Array

### Array 특징
  1. <span style="color:red">인덱스</span>가 존재
     - 제한된 범위가 있는 정수
     - 배열은 $$ 2^{32} $$-1개 까지의 요소를 가질 수 있다.
     - 즉 0부터 $$ 2^{32} $$-2 까지의 범위만 유효한 인덱스 값이다.
  2. <span style="color:red">길이 정보</span>가 존재
     - 배열에 요소가 추가될 경우 `length property`가 증가

### Array 함수
  - `pop`
    - 배열 뒷 부분 값을 삭제 

      ```javascript
      var arr = [1, 2]
      arr.pop()
      # arr: [1]
      ```

  - `push`
    - 배열 뒷 부분에 값을 삽입

      ```javascript
      var arr = [1, 2]
      arr.push(3)
      # arr: [1, 2, 3]
      ```

  - `shift`
    - 배열 앞 부분 값 삭제

    ```javascript
    var arr = [1, 2, 3];
    arr.shift();
    #arr: [2, 3]
    ```

  - `unshift`
    - 배열 앞 부분에 값 삽입

    ```javascript
    var arr = [1, 2, 3];
    arr.unshift( 0 );
    # [0, 1, 2, 3]
    ```

  - `concat`
    - 다수의 배열을 합치고 병합한 배열의 사본 반환

    ```javascript
    var arr1 = [ 1, 2, 3 ];
    var arr2 = [ 4, 5, 6 ];
    var arr3 = arr2.concat( arr1 );
    # [4, 5, 6, 1, 2, 3 ]
    ```

  - `reverse`
    - 배열의 요소 순서를 거꾸로 변경

    ```javascript
    var arr =[1, 2, 3];
    arr.reverse()
    # [3, 2, 1]
    ```

  - `toString`
    - 배열 요소를 하나의 문자열로 합친다.
    - join 함수에 인자 안주고 실행하는 것과 결과 동일

    ```javascript
    var arr = [1, 2, 3];
    arr.toString(); 
    # 1,2,3
    ```

  - `join`
    - 배열 요소 전부를 하나의 문자열로 합친다.

    ```javascript
    var arr = [1, 2, 3];
    arr.join(); 
    # 1,2,3
    
    arr.join('@')
    # 1@2@3

---

## 레퍼런스

- https://velog.io/@godori/JavaScript-engine-1
- https://velog.io/@godori/JavaScript-엔진-톺아보기-2-pujpqum2ji
- https://mathiasbynens.be/notes/shapes-ics
- http://blog.302chanwoo.com/2017/08/javascript-array-method/

---
