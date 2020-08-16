---
title: [번역] play-with-spidermonkey
categories: Browser
tags: [theory, firefox]
---

# 목차

- Representing Values
  - JSValue
  - JSObject
    - group_
    - shape_ and slots_
    - elements_
- Typed Array
- References

# Representing Values

## JS Value

자바 스크립트는 "type"을 정의하지 않고 변수에 값을 할당할 수 있다.  그렇다면 자바스크립트는 변수의 type을 어떻게 추적할 수 있을까?

데이터의 모든 "type"은 `JS::Value`의 객체로 표시된다. JS::Value 또는 jsval은 "type"과  "value"를 한 단위로 인코딩하여 다양한 type을 나타낸다. jsval에서 **상위 17bit는 jsval의 type을 나타내는 태그**이며 **하위 47bit는 실제 값**이다.

예를 들어보자. 다음과 같이 js shell을 실행하고 다른 type의 값을 저장하는 배열을 만들었다.

```jsx
js> a=[0x11223344, "STRING", 0x44332211, true]
[287454020, "STRING", 1144201745, true]
```

현재 배열은 int, string, int, boolean 순서로 구성되어 있다. 이제 gdb를 attach해서 값이 어떻게 보이는지 볼 것이다.

`ps -ef` 명령으로 pid를 확인하고 pwndbg에 attach를 하였다. 그리고 값을 확인해보면 다음과 같이 저장되어 있다.

```jsx
pwndbg> search -t dword 0x11223344
                0x7f6103f44af0 0xfff8800011223344
pwndbg> x/4gx 0x7f6103f44af0
0x7f6103f44af0:	0xfff8800011223344	0xfffaff6104035c40
0x7f6103f44b00:	0xfff8800044332211	0xfff9800000000001
```

int형 0x11223344는 0xfff8800011223344로 저장된다. 다음의 코드는 `js/public/Value.h`이다.

```c
enum JSValueType : uint8_t
{
    JSVAL_TYPE_DOUBLE              = 0x00,
    JSVAL_TYPE_INT32               = 0x01,
    JSVAL_TYPE_BOOLEAN             = 0x02,
    JSVAL_TYPE_UNDEFINED           = 0x03,
    JSVAL_TYPE_NULL                = 0x04,
    JSVAL_TYPE_MAGIC               = 0x05,
    JSVAL_TYPE_STRING              = 0x06,
    JSVAL_TYPE_SYMBOL              = 0x07,
    JSVAL_TYPE_PRIVATE_GCTHING     = 0x08,
    JSVAL_TYPE_OBJECT              = 0x0c,

    /* These never appear in a jsval; they are only provided as an out-of-band value. */
    JSVAL_TYPE_UNKNOWN             = 0x20,
    JSVAL_TYPE_MISSING             = 0x21
};

----

JS_ENUM_HEADER(JSValueTag, uint32_t)
{
    JSVAL_TAG_MAX_DOUBLE           = 0x1FFF0,
    JSVAL_TAG_INT32                = JSVAL_TAG_MAX_DOUBLE | JSVAL_TYPE_INT32,
    JSVAL_TAG_UNDEFINED            = JSVAL_TAG_MAX_DOUBLE | JSVAL_TYPE_UNDEFINED,
    JSVAL_TAG_NULL                 = JSVAL_TAG_MAX_DOUBLE | JSVAL_TYPE_NULL,
    JSVAL_TAG_BOOLEAN              = JSVAL_TAG_MAX_DOUBLE | JSVAL_TYPE_BOOLEAN,
    JSVAL_TAG_MAGIC                = JSVAL_TAG_MAX_DOUBLE | JSVAL_TYPE_MAGIC,
    JSVAL_TAG_STRING               = JSVAL_TAG_MAX_DOUBLE | JSVAL_TYPE_STRING,
    JSVAL_TAG_SYMBOL               = JSVAL_TAG_MAX_DOUBLE | JSVAL_TYPE_SYMBOL,
    JSVAL_TAG_PRIVATE_GCTHING      = JSVAL_TAG_MAX_DOUBLE | JSVAL_TYPE_PRIVATE_GCTHING,
    JSVAL_TAG_OBJECT               = JSVAL_TAG_MAX_DOUBLE | JSVAL_TYPE_OBJECT
} JS_ENUM_FOOTER(JSValueTag);

----

enum JSValueShiftedTag : uint64_t
{
    JSVAL_SHIFTED_TAG_MAX_DOUBLE      = ((((uint64_t)JSVAL_TAG_MAX_DOUBLE)     << JSVAL_TAG_SHIFT) | 0xFFFFFFFF),
    JSVAL_SHIFTED_TAG_INT32           = (((uint64_t)JSVAL_TAG_INT32)           << JSVAL_TAG_SHIFT),
    JSVAL_SHIFTED_TAG_UNDEFINED       = (((uint64_t)JSVAL_TAG_UNDEFINED)       << JSVAL_TAG_SHIFT),
    JSVAL_SHIFTED_TAG_NULL            = (((uint64_t)JSVAL_TAG_NULL)            << JSVAL_TAG_SHIFT),
    JSVAL_SHIFTED_TAG_BOOLEAN         = (((uint64_t)JSVAL_TAG_BOOLEAN)         << JSVAL_TAG_SHIFT),
    JSVAL_SHIFTED_TAG_MAGIC           = (((uint64_t)JSVAL_TAG_MAGIC)           << JSVAL_TAG_SHIFT),
    JSVAL_SHIFTED_TAG_STRING          = (((uint64_t)JSVAL_TAG_STRING)          << JSVAL_TAG_SHIFT),
    JSVAL_SHIFTED_TAG_SYMBOL          = (((uint64_t)JSVAL_TAG_SYMBOL)          << JSVAL_TAG_SHIFT),
    JSVAL_SHIFTED_TAG_PRIVATE_GCTHING = (((uint64_t)JSVAL_TAG_PRIVATE_GCTHING) << JSVAL_TAG_SHIFT),
    JSVAL_SHIFTED_TAG_OBJECT          = (((uint64_t)JSVAL_TAG_OBJECT)          << JSVAL_TAG_SHIFT)
};
```

type은 `JSValueType`에 표시된 숫자와  `JSVAL_TAG_MAX_DOUBLE`의 값인 0x1fff0과 or 연산 된 값을 47비트만큼 오른쪽으로 쉬프트해서 만든다. 이렇게하면 총 17비트의 태그가 만들어진다.

예를 들어 int에 대한 태그는 다음과 같이 만들어진다.

```c
(1 | 0x1FFF0) << 47 = 0xfff8800000000000
```

## JSObject

자바스크립트는 배열과 같은 다양한 type의 object도 가지고 있다.  object는 "property"를 가지고 있다.

```c
obj = { p1: 0x11223344, p2: "STRING", p3: true, p4: [1.2,3.8]};
```

위의 예시에서 p1, p2, p3, p4는 obj의 "property"이다. 이건 파이썬의 딕셔너리와 같다. 각각의 property는 매핑된 값을 가지고 있다. 이 값은 int, string, boolean, object 등 모든 type일 수 있다. 이러한 object는 메모리에 JSObject class의 object로 표시된다.

다음은 NativeObject class의 추상화로서, JSObject를 상속한다.

```cpp
class NativeObject
{
    js::GCPtrObjectGroup group_;
    void* shapeOrExpando_;
    js::HeapSlot *slots_;
    js::HeapSlot *elements_;
};
```

### group_

`js/src/vm/JSObject.h`에는 다음과 같은 주석이 있다.

> group_ 멤버는 prototype object, class, property의 가능한 type들을 포함하는 object의 group을 저장한다.

group_ 멤버는 본질적으로 ObjectGroup class의 멤버로 다음과 같은 멤버를 가지고 있다.

```cpp
    /* Class shared by objects in this group. */
    const Class* clasp_; // set by constructor

    /* Prototype shared by objects in this group. */
    GCPtr<TaggedProto> proto_; // set by constructor

    /* Realm shared by objects in this group. */
    JS::Realm* realm_;; // set by constructor

    /* Flags for this group. */
    ObjectGroupFlags flags_; // set by constructor

    // If non-null, holds additional information about this object, whose
    // format is indicated by the object's addendum kind.
    void* addendum_ = nullptr;

    Property** propertySet = nullptr;
```

주석들은 각 필드의 용도를 설명해주는데, 일단 `clasp_`를 볼 것이다. clasp_는 익스플로잇할 때 유용하게 이용할 수 있다.

주석을 보면 이것은 이 group에 속한 모든 Object가 공유하는 JSClass를 정의하며 이 group을 식별하는 데 사용할 수 있다. 이제 클래스 구조체를 보자,

```cpp
struct MOZ_STATIC_CLASS Class
{
    JS_CLASS_MEMBERS(js::ClassOps, FreeOp);
    const ClassSpec* spec;
    const ClassExtension* ext;
    const ObjectOps* oOps;
    :
    :
}
```

ClassOps는 기본적으로 object에 대한 특정 동작을 정의하는 다양한 함수 포인터들의 구조체를 가리키는 포인터이다. ClassOps 구조체는 다음과 같다.

```cpp
struct MOZ_STATIC_CLASS ClassOps
{
    /* Function pointer members (may be null). */
    JSAddPropertyOp     addProperty;
    JSDeletePropertyOp  delProperty;
    JSEnumerateOp       enumerate;
    JSNewEnumerateOp    newEnumerate;
    JSResolveOp         resolve;
    JSMayResolveOp      mayResolve;
    FinalizeOp          finalize;
    JSNative            call;
    JSHasInstanceOp     hasInstance;
    JSNative            construct;
    JSTraceOp           trace;
};
```

예를들어 `addProperty` 필드의 함수 포인터는 새로운 property를 호출할 때 호출할 함수를 정의한다.  우리는 기본적으로 함수 포인터의 배열을 가지고 있다. 만약 우리가 이 함수 포인터 중 하나를 덮어쓰는데 성공한다면 임의 주소 쓰기(aaw)를 임의 코드 실행으로 만들 수 있다.

하지만 쉽지는 않다. 문제는 함수 포인터가 있는 영역이 `r-x`(쓰기 권한이 없음) 영역이라는 점이다. 그러나 aaw가 있다면 전체 `ClassOps` 구조체를 쉽게 변조할 수 있고 `group` 필드의 실제 `ClassOps`에 대한 포인터를 가짜 구조체에 대한 포인터로 변조할 수 있다. (FSOP 떠올리면 될 듯)

따라서 aaw가 있다면 프로그램 흐름 변조를 할 수 있는 수단이 여기에 있고 이는 나중에 쓸모 있을 것이다.

### shape_ and slots_

자바스크립트는 object의 property를 어떻게 알 수 있을까?  다음의 예시를 보자.

```cpp
obj = {}
obj.blahblah = 0x55667788
obj.strtest = "TESTSTRING"
```

obj는 배열이지만 몇 가지의 property도 가지고 있다. 이제 자바스크립트는 값 뿐만 아니라 property 이름까지 알아내야 한다. 이를 위해서 object 필드의  shape_ 와 slots_를 사용한다.

slots_ 필드는 각 property에 대한 값을 포함하는 필드이다. 기본적으로 값만 포함하는 배열이다. shape_에는 property의 이름과 이 peoperty의 값이 저장되어 있는 slots_ 배열의 인덱스가 저장되어 있다.

즉 아래의 그림을 보면 이해하기 편할 것이다.

![Untitled](https://user-images.githubusercontent.com/43925259/90337063-f4db3680-e01a-11ea-8227-b61b3929d03b.png)

### elements_

이전 섹션에서 살펴본 예제에서는 object는 몇가지 property를 가지고 있을 뿐이었다. 이제 element를 추가해볼 것이다. 다음의 코드를 기존의 코드에 추가하자.

```jsx
obj[0]=0x11223344
obj[1]=0x33557711
```

element들은 `elements_` 멤버를 가리키는 배열로 저장될 것이다. 수정된 이미지는 다음과 같다.

![Untitled (1)](https://user-images.githubusercontent.com/43925259/90337092-353ab480-e01b-11ea-981e-ebdeb143c6fe.png)

이제 object에 특정 번호의 elements를 추가할 수 있다는 것을 알게 되었다. 그래서 elements 배열에는 번호를 알아내기 위한 메타데이터가 있다. 자세한 내용은 `js/src/vm/NativeObject.h`를 확인하면 된다. 메타데이터는 다음과 같다.

```cpp
uint32_t flags;

/*
 * Number of initialized elements. This is <= the capacity, and for arrays
 * is <= the length. Memory for elements above the initialized length is
 * uninitialized, but values between the initialized length and the proper
 * length are conceptually holes.
 */
uint32_t initializedLength;

/* Number of allocated slots. */
uint32_t capacity;

/* 'length' property of array objects, unused for other objects. */
uint32_t length;
```

위의 코드는 NativeObject.h의 ObjectElements 정의에 있다.  elements_의 시작 주소로부터 0x10을 뺀 영역에 순서대로 flags, init_len, capacity, length가 존재한다.

# Typed Array

다음은 MDN 페이지의 내용이다.

> `ArrayBuffer`는 일반적인 고정 길이 데이터를 나타낼 때 사용하는 데이터 유형이다. `ArrayBuffer`의 내용을 직접 조작할 수 없는 대신 특정 형식으로 버퍼를 나타내는 `DataView` 또는 typed array view를 생성하여 버퍼의 내용을 읽고 쓰는데 사용할 수 있다.

`NativeObject`의 모든 속성은 `ArrayBufferObject`에 의해 상속된다.  `ArrayBufferObject`는 다음과 같다.

- **Pointer to data**: private 형식의 ArrayBuffer의 데이터를 가리키는 포인터
- **length**: 버퍼의 크기
- **Frist View**: 현재 ArrayBuffer를 참조하는 first view를 가리키는 포인터
- **flags**

이제 다음과 같은 코드를 작성해보자.

```jsx
arrbuf = new ArrayBuffer(0x100);        // ArrayBuffer of size 0x100 bytes.
uint32view = new Uint32Array(arrbuf);   // Adding a Uint32 view.
uint16view = new Uint16Array(arrbuf);   // Adding another view - this time a Uint16 one.
uint32view[0]=0x11223344                // Initialize the buffer with a value.

uint32view[0].toString(16)
// Outputs "11223344"

/* Lets check the Uint16Array */

uint16view[0].toString(16)
// Outputs "3344"

uint16view[1].toString(16)
// Outputs "1122"
```

동일한 버퍼에 대한 다른 view는 버퍼의 데이터를 다른 방식으로 볼 수 있게 해준다. `ArrayBuffer` 와 같은 `TypeArray`는 `NativeObject`외에도 추가 속성을 가지고 있다.

- **Underlying ArrayBuffer**: typed array의 데이터를 저장하는 ArrayBuffer에 대한 포인터
- **length**: 배열의 길이 (만약 ArrayBuffer가 32byte이고 Uint32Array인 경우 길이는 32/4 = 8이다.)
- **offset**
- **pointer to data**: 성능 향상을 위한 raw 형태의 데이터 버퍼에 대한 포인터

Typed Array는 임의의 위치에서 데이터를 읽고 쓰는 것이 필요할 때 매우 유용하다. ArrayBuffer의 데이터 포인터를 제어할 수 있다면 Uint32Array를 할당하여 임의의 위치에서 한 번에 4바이트를 읽고 쓸 수 있다. 대신에 일반 배열을 사용한다면 임의의 위치에서 읽은 데이터는 float일 것이고 데이터를 쓰기 위해 페이로드를 int 대신 float로 작성해야 한다.

# References

https://vigneshsrao.github.io/play-with-spidermonkey/
