# claujson-document
# first check!
    01. 64bit only
    02. need fast memory allocator like mimalloc, for speed.
    03. C++14~ 
    04. experimental!
    05. Array has std::vector<_Value>
    06. Object has std::vector<Pair<_Value, _Value>> (not a std::map!)
    07. in Object, key can be dupplicated, are not sorted, just in order of input.
    08. scanning - modified? simdjson, parsing - parallel 
    09. it is not read-only! 
    10. _Value <- json_Value, (in destructor, no remove data(Array or Object!), no copy, only move or clone!
    11. Value <- wrap _Value, (in destructor, remove data(Array or Object!)
    12. Array, Object <- in destructor, remove data.

# claujson 라이브러리 기술 문서 - using claude

> C++17 기반 고성능 멀티스레드 JSON 파서 라이브러리

---

## 목차

1. [개요](#1-개요)
2. [아키텍처 및 핵심 클래스](#2-아키텍처-및-핵심-클래스)
3. [클래스 상세 설명](#3-클래스-상세-설명)
4. [파싱 흐름](#4-파싱-흐름)
5. [직렬화 흐름](#5-직렬화-흐름)
6. [주요 API](#6-주요-api)
7. [내부 구현 세부사항](#7-내부-구현-세부사항)
8. [주의사항 및 제약](#8-주의사항-및-제약)

---

## 1. 개요

claujson은 C++17로 작성된 고성능 JSON 파서 라이브러리입니다. 내부적으로 **simdjson의 SIMD 기반 스테이지 1 토크나이저**를 재활용하며, 멀티스레드 병렬 파싱과 직렬화를 지원합니다.

### 주요 특징

- simdjson 기반 SIMD 가속 토크나이징 (Stage 1)
- 멀티스레드 병렬 파싱 및 직렬화 (`ThreadPool` 사용)
- JSON 파일 및 문자열 파싱 지원
- JSON Pointer(`json_pointerB`) 지원
- `diff` / `patch` 기능 내장
- **Short String Optimization**: 11바이트 이하 문자열은 힙 할당 없음
- UTF-8 유효성 검사 통합
- C++20 `char8_t` (`u8string_view`) 지원

### 지원 환경

| 항목 | 내용 |
|------|------|
| C++ 표준 | C++17 이상 (char8_t 사용 시 C++20) |
| 아키텍처 | **64비트 전용** (32비트 빌드 불가) |
| 의존성 | simdjson (수정본), C++ 표준 라이브러리 |
| 스레딩 | `std::thread` 기반 `ThreadPool` |

---

## 2. 아키텍처 및 핵심 클래스

### 클래스 구조 개요

```
Document
└── _Value  (루트 JSON 값)
      ├── Array*   → Array  (arr_vec: vector<_Value>)
      │                └── _Value (재귀)
      ├── Object*  → Object (obj_data: vector<Pair<_Value, _Value>>)
      │                └── Pair<key:_Value, value:_Value> (재귀)
      └── 원시값  (int64 / uint64 / double / bool / null / String)
```

### 파일 구성

| 파일 | 역할 |
|------|------|
| `claujson.h / .cpp` | 진입점, `parser`, `writer`, `StructuredPtr`, 전역 유틸 |
| `claujson_internal.h` | 공용 타입(`_ValueType`, `Pair`, `Vector`, `Log`) |
| `claujson_string.h` | `String` (SSO 문자열 클래스) |
| `claujson_value.cpp` | `_Value` 메서드 구현 |
| `claujson_array.h / .cpp` | `Array` 클래스 |
| `claujson_object.h / .cpp` | `Object` 클래스 |
| `claujson_partialjson.h / .cpp` | `PartialJson` (멀티스레드 파싱 중간 결과) |

---

## 3. 클래스 상세 설명

### 3.1 `_Value`

모든 JSON 값을 표현하는 **핵심 타입**입니다. 복사 생성/대입이 삭제되어 있으며, 이동만 허용됩니다. 명시적 복사가 필요할 때는 `clone()`을 사용합니다.

#### 내부 메모리 레이아웃

```cpp
union {
    struct {
        union {
            int64_t  _int_val;
            uint64_t _uint_val;
            double   _float_val;
            Array*   _array_ptr;
            Object*  _obj_ptr;
            PartialJson* _pj_ptr;
            bool     _bool_val;
        };
        uint32_t   temp;
        _ValueType _type;   // 타입 구분자
    };
    String _str_val;        // STRING / SHORT_STRING일 때 사용
};
```

> `_type`이 `STRING` 또는 `SHORT_STRING`일 때만 `_str_val`에 접근해야 합니다. 두 분기는 같은 메모리를 공유합니다.

#### `_ValueType` 열거형

| 값 | 의미 |
|----|------|
| `NONE` | 초기화되지 않은 상태 |
| `ARRAY` | JSON 배열 (`_array_ptr` 유효) |
| `OBJECT` | JSON 객체 (`_obj_ptr` 유효) |
| `PARTIAL_JSON` | 파싱 중간 결과 (내부용) |
| `INT` | 64비트 부호 있는 정수 |
| `UINT` | 64비트 부호 없는 정수 |
| `FLOAT` | 64비트 부동소수점 (double) |
| `BOOL` | 불리언 |
| `NULL_` | JSON null |
| `STRING` | 동적 할당 문자열 (길이 ≥ 11) |
| `SHORT_STRING` | 스택 내 인라인 문자열 (길이 < 11, SSO) |
| `NOT_VALID` | 유효하지 않은 값 (오류 반환 시) |
| `ERROR` | 내부 오류 상태 |

#### 주요 메서드

```cpp
// 타입 확인
bool is_valid() const;
bool is_null() const;
bool is_primitive() const;   // int/uint/float/bool/string/null
bool is_structured() const;  // array 또는 object
bool is_array() const;
bool is_object() const;
bool is_int() const;
bool is_uint() const;
bool is_float() const;
bool is_number() const;      // int || uint || float
bool is_bool() const;
bool is_str() const;

// 값 접근 (getter)
int64_t  int_val() const;
uint64_t uint_val() const;
double   float_val() const;
bool     bool_val() const;
String&  str_val();

// 구조체 접근
Array*      as_array();
Object*     as_object();
StructuredPtr as_structured_ptr();

// 값 설정 (setter)
void set_int(long long x);
void set_uint(unsigned long long x);
void set_float(double x);
bool set_str(const char* str, uint64_t len);
void set_bool(bool x);
void set_null();

// 인덱스/키 접근
_Value& operator[](uint64_t idx);
_Value& operator[](const _Value& key);

// JSON Pointer 탐색
_Value& json_pointerB(const std_vector<_Value>& routeVec);

// 명시적 복사
_Value clone() const;
```

---

### 3.2 `Value`

`_Value`를 소유하는 **RAII 래퍼 클래스**입니다. 소멸 시 `claujson::clean()`을 자동 호출하여 하위 구조체 메모리를 해제합니다. 복사는 불가하며 이동만 허용됩니다.

```cpp
Value v(Array::Make());        // _Value를 Value로 래핑
_Value& inner = v.Get();       // 내부 _Value 접근
```

---

### 3.3 `Document`

파싱 결과를 담는 **최상위 컨테이너**입니다. `parser::parse()` 또는 `parser::parse_str()`의 출력 대상입니다.

```cpp
claujson::parser p;
claujson::Document doc;
auto [ok, len] = p.parse("data.json", doc, 0 /*auto thread*/);
_Value& root = doc.Get();
```

---

### 3.4 `Array`

JSON 배열을 나타내며, `std::vector<_Value>` (`arr_vec`)로 원소를 저장합니다.

#### 주요 멤버

```cpp
std_vector<_Value> arr_vec;  // 원소 저장
Pointer parent;              // 부모 포인터 (비트 패킹)
```

#### 주요 메서드

```cpp
static _Value Make();                    // 빈 배열 _Value 생성
static _Value MakeVirtual();             // 가상(virtual) 배열 생성 (멀티스레드 내부용)

bool add_element(Value val);             // 원소 추가 (말미)
bool assign_element(uint64_t idx, Value val); // 원소 교체
bool insert(uint64_t idx, Value val);    // 중간 삽입
void erase(uint64_t idx, bool real = false);  // 원소 삭제

uint64_t get_data_size() const;
_Value& get_value_list(uint64_t idx);

uint64_t find(const _Value& value, uint64_t start = 0) const;
StructuredPtr get_parent() const;
bool is_virtual() const;
```

> `erase(idx, real=true)`이면 원소에 `clean()`을 호출하여 하위 구조체까지 재귀 해제합니다.

---

### 3.5 `Object`

JSON 객체를 나타내며, `std::vector<Pair<_Value, _Value>>` (`obj_data`)로 키-값 쌍을 순서대로 저장합니다. (비정렬 맵이 아닌 **순서 보존 벡터** 구조)

#### 주요 메서드

```cpp
static _Value Make();
static _Value MakeVirtual();

bool add_element(Value key, Value val);
bool assign_value_element(uint64_t idx, Value val);
void erase(const _Value& key, bool real = false);
void erase(uint64_t idx, bool real = false);
bool change_key(const _Value& key, Value new_key);
bool change_key(uint64_t idx, Value new_key);

uint64_t find(const _Value& key) const;   // 선형 탐색 O(n)
bool chk_key_dup(uint64_t* idx) const;    // 중복 키 검사

_Value& get_key_list(uint64_t idx);
_Value& get_value_list(uint64_t idx);
```

> `find()`는 내부적으로 선형 탐색을 수행합니다 (O(n)). 대규모 객체에서 빈번한 키 탐색은 성능에 영향을 줄 수 있습니다.

---

### 3.6 `StructuredPtr`

`Array*`, `Object*`, `PartialJson*` 중 하나를 가리키는 **타입 이레이저(type-erased) 포인터**입니다. `type` 필드로 실제 타입을 구분합니다.

```cpp
union { Array* arr; Object* obj; PartialJson* pj; };
uint32_t type;  // 0: null, 1: Array, 2: Object, 3: PartialJson
```

파싱 엔진(`LoadData2`)에서 Array/Object/PartialJson을 동일한 인터페이스로 다루기 위해 사용합니다.

---

### 3.7 `String` (Short String Optimization)

11바이트(`CLAUJSON_STRING_BUF_SIZE`) 이하 문자열은 힙 할당 없이 인라인 버퍼에 저장합니다.

```
┌─────────────────────────────────────────────┐
│ SHORT_STRING (len < 11)                     │
│   buf[11] + buf_sz(uint8) + type_(_ValueType)│
├─────────────────────────────────────────────┤
│ STRING (len >= 11)                          │
│   char* str + uint32_t sz + _ValueType type  │
└─────────────────────────────────────────────┘
```

복사 생성자는 `protected`로 제한되어 있으며, `_Value`만 직접 생성할 수 있습니다. 외부에서는 `_Value::clone()`을 통해 간접 복사합니다.

---

### 3.8 `Pointer` (비트 패킹 부모 포인터)

`Array`와 `Object`의 `parent` 필드에 사용됩니다. 64비트 포인터 하나에 세 가지 정보를 인코딩합니다.

```
bit 63      : is_virtual 플래그 (1이면 가상 노드)
bits 62~2   : 실제 부모 포인터 주소
bits 1~0    : 부모 타입 (1: Array, 2: Object, 3: PartialJson)
```

포인터를 사용할 때는 `use()` 메서드로 비트 마스크를 제거한 순수 주소를 얻습니다.

---

### 3.9 `PartialJson` (내부용)

멀티스레드 파싱 중 각 스레드가 생산하는 **부분 파싱 결과**를 저장하는 클래스입니다. 외부 사용자가 직접 생성하거나 사용할 필요는 없습니다.

배열 파편과 객체 파편을 각각 `arr_vec`, `obj_data`에 저장하고, 스레드 경계에 걸쳐 있는 가상 노드는 `virtualJson`에 저장합니다.

---

### 3.10 `parser` / `writer`

```cpp
// 파싱
claujson::parser p(thr_num);  // thr_num=0이면 자동 결정
auto [ok, len] = p.parse("file.json", doc, thr_num);
auto [ok, len] = p.parse_str("{ \"key\": 1 }", doc, thr_num);

// 직렬화
claujson::writer w(thr_num);
std::string s  = w.write_to_str(doc.Get(), /*pretty=*/false);
std::string s2 = w.write_to_str2(doc.Get(), false);   // JsonView 기반 방식
w.write("output.json", doc.Get(), false);
w.write_parallel("output.json", doc.Get(), thr_num, false);
w.write_parallel2("output.json", doc.Get(), thr_num, false);
```

---

## 4. 파싱 흐름

```
입력 (파일 / 문자열)
        │
        ▼
[Stage 1: simdjson]  토크나이징 → structural_indexes 배열 생성
        │
        ▼
[is_valid2]  병렬 구조 검증 + count_vec(각 노드 자식 수) 계산
        │
        ▼
[FindDivisionPlace]  comma 기준으로 스레드 분할 경계(pivot) 탐색
        │
        ├─ Thread 0 ─► __LoadData(pivot[0] ~ pivot[1]) → PartialJson 0
        ├─ Thread 1 ─► __LoadData(pivot[1] ~ pivot[2]) → PartialJson 1
        └─ Thread N ─► __LoadData(pivot[N-1] ~ pivot[N]) → PartialJson N
                │
                ▼
        [Merge]  PartialJson들을 순서대로 병합
                │
                ▼
        Document::Get() 에 최종 _Value 저장
```

### `__LoadData` 내부 동작

simdjson이 생성한 `structural_indexes`를 순회하면서 아래 상태 머신을 실행합니다.

| 토큰 | 동작 |
|------|------|
| `{` | 새 `Object` 생성, `nowUT` 갱신 |
| `[` | 새 `Array` 생성, `nowUT` 갱신 |
| `}` / `]` (brace > 0) | `nowUT`를 부모로 이동 |
| `}` / `]` (brace == 0) | 스레드 경계 처리: 가상(virtual) 노드 생성 |
| `key : value` | `add_item_type()` 호출 |
| `value` (배열 원소) | `add_item_type()` 호출 |
| `,` | 스킵 |

### Merge 알고리즘

각 스레드의 `PartialJson`은 `Merge()` / `Merge2()`를 통해 순서대로 병합됩니다.

```
next (앞 청크 끝 노드)  +  ut (다음 청크 가상 노드)
    → MergeWith()로 ut의 원소들을 next에 이동
    → 부모 포인터를 따라 루트까지 반복
```

---

## 5. 직렬화 흐름

claujson은 두 가지 직렬화 방식을 제공합니다.

### 방식 1: 재귀 순회 (`_write`)

`LoadData2::_write()`가 `_Value` 트리를 재귀적으로 순회하며 `StrStream`에 출력합니다. 단일 스레드 방식과 병렬 방식(`write_parallel`) 모두 이 방식을 사용합니다.

병렬 직렬화 시:
1. `Divide2()`로 트리를 n개 청크로 분할
2. 각 청크를 별도 스레드에서 `write_()`로 직렬화
3. 결과 `StrStream`들을 순서대로 파일에 기록
4. 분할된 트리는 `Merge2()`로 원상복구

### 방식 2: JsonView 기반 (`write_to_str2`, `write_parallel2`)

1. `Size2()`로 전체 노드 수 계산
2. `run()`으로 트리를 `JsonView[]` 배열(선형화)로 변환
3. 배열을 n 구간으로 나눠 `print()` / `print_pretty()`를 병렬 실행

`JsonView` 타입 코드:

| type | 의미 |
|------|------|
| 0 | ARRAY 시작 |
| 1 | OBJECT 시작 |
| 2 | KEY |
| 3 | VALUE (원시값) |
| 4 | ARRAY 끝 |
| 5 | OBJECT 끝 |
| -1 | 종료 마커 |

---

## 6. 주요 API

### 파싱

```cpp
#include "claujson.h"

claujson::parser parser(/*thr_num=*/0); // 0: 자동

// 파일 파싱
claujson::Document doc;
auto [ok, len] = parser.parse("data.json", doc, 0);

// 문자열 파싱
auto [ok2, len2] = parser.parse_str(R"({"name":"claujson","version":1})", doc, 0);

// C++20 char8_t
auto [ok3, len3] = parser.parse_str(u8R"({"hello":"world"})", doc, 0);
```

### 값 접근

```cpp
_Value& root = doc.Get();

// 배열 접근
if (root.is_array()) {
    uint64_t sz = root.as_array()->size();
    for (uint64_t i = 0; i < sz; ++i) {
        _Value& elem = root[i];
        if (elem.is_int())   std::cout << elem.get_integer();
        if (elem.is_str())   std::cout << elem.str_val().data();
    }
}

// 객체 접근
if (root.is_object()) {
    _Value key = _Value("name"sv);
    _Value& val = root[key];
    if (val.is_str()) std::cout << val.str_val().data();
}
```

### JSON Pointer

```cpp
// /users/0/name 경로 탐색
std_vector<_Value> route;
route.push_back(_Value("users"sv));
route.push_back(_Value((uint64_t)0));
route.push_back(_Value("name"sv));

_Value& target = root.json_pointerB(route);
```

### 값 수정

```cpp
// 배열에 원소 추가
StructuredPtr sp(root.as_array());
sp.add_array_element(_Value((int64_t)42));

// 객체에 키-값 추가
StructuredPtr so(root.as_object());
so.add_object_element(_Value("score"sv), _Value(99.5));

// 원소 삭제
sp.erase((uint64_t)0, /*real=*/true);
so.erase(_Value("score"sv), true);
```

### 직렬화

```cpp
claujson::writer writer(0);

// 문자열로 직렬화
std::string json_str = writer.write_to_str(doc.Get(), /*pretty=*/false);

// 파일로 저장
writer.write("output.json", doc.Get(), false);

// 병렬 직렬화 (대용량)
writer.write_parallel("output.json", doc.Get(), /*thr_num=*/0, false);
writer.write_parallel2("output.json", doc.Get(), 0, false);
```

### diff / patch

```cpp
_Value a = /* ... */;
_Value b = /* ... */;

_Value d = claujson::diff(a, b);     // 차이 계산 (JSON Patch 유사 형식)
claujson::patch(a, d);               // a에 패치 적용 → b와 동일해짐
```

### 유틸리티

```cpp
// 메모리 해제 (Array/Object 재귀 삭제)
claujson::clean(value);
/*
// 문자열 유효성 검사
bool ok = claujson::is_valid_string_in_json("hello\\nworld");

// JSON 이스케이프 변환
auto [ok, converted] = claujson::convert_to_string_in_json("hello\\nworld");

// 숫자 파싱
_Value num;
claujson::convert_number("3.14", num);
*/
// 로그 설정
claujson::log.console();
claujson::log.info();
claujson::log.warn();
```

---

## 7. 내부 구현 세부사항

### `_Value` 이동 시맨틱

```cpp
_Value::_Value(_Value&& other) noexcept {
    if (other.is_str()) {
        _str_val = std::move(other._str_val);  // String 이동
    } else {
        std::swap(_int_val, other._int_val);
        std::swap(_type, other._type);         // 타입과 값만 스왑
    }
}
```

`operator=(_Value&&)`는 `std::swap` 후 `clean(other)`를 호출하여 이전 값을 명시적으로 정리합니다.

### 문자열 파싱 (`set_str`)

외부에서 입력되는 문자열은 `simdjson::parse_string()`을 통해 JSON 이스케이프 시퀀스(`\n`, `\uXXXX` 등)를 디코딩하고, `validate_utf8()`로 UTF-8 유효성을 검사합니다. 1024바이트를 기준으로 스택 버퍼와 힙 버퍼를 분기하여 처리합니다.

파싱 중에는 이 과정을 생략하고 `set_str_in_parse()`(simdjson이 이미 처리한 결과를 직접 저장)를 사용합니다.

### `count_vec`와 `reserve_data_list`

`is_valid2()`는 구조 검증과 동시에 각 Array/Object가 가진 직접 자식 원소 수를 `count_vec`에 기록합니다. `__LoadData()`는 `{` / `[` 진입 시 이 값으로 `reserve_data_list()`를 호출하여 벡터 재할당을 최소화합니다.

### `clean()` 함수

```cpp
void clean(_Value& x) {
    if (x.is_array())        delete x.as_array();
    else if (x.is_object())  delete x.as_object();
    else if (x.is_partial_json()) delete x.as_partial_json();
    x.set_none();
}
```

`Array`와 `Object`의 소멸자가 하위 구조체 포인터를 재귀 삭제하므로, `clean()`은 최상위 노드 하나만 `delete`하면 전체 트리가 정리됩니다.

---

## 8. 주의사항 및 제약

| 항목 | 내용 |
|------|------|
| **64비트 전용** | `Pointer` 클래스가 64비트 포인터 비트 조작에 의존합니다. 32비트 빌드는 불가합니다. |
| `_Value` 복사 불가 | 복사 생성자와 복사 대입 연산자가 `delete`되어 있습니다. 복사가 필요하면 `clone()`을 사용하세요. |
| `Object::find()` 성능 | 선형 탐색(O(n))입니다. 매우 큰 객체에서 반복 키 검색은 피하세요. |
| `PartialJson` 직접 사용 금지 | 파싱 엔진 내부용 클래스이며, 외부에서 직접 생성하거나 조작해서는 안 됩니다. |
| `String` 복사 생성자 | `protected`로 제한되어 있어 `_Value` 외부에서는 복사 생성이 불가합니다. |
| `is_valid()` 확인 | `NOT_VALID` 또는 `ERROR` 상태의 `_Value`에 값 접근 시 미정의 동작이 발생할 수 있습니다. 항상 `is_valid()`를 먼저 확인하세요. |
| `clean()` 중복 호출 | `clean()` 후 해당 `_Value`로 `delete`를 다시 호출하면 이중 해제(double-free)가 발생합니다. `Document`와 `Value` 소멸자가 자동으로 처리하므로 수동 호출에 주의하세요. |
| 멀티스레드 안전성 | 파싱/직렬화 자체는 내부적으로 스레드를 관리하지만, `_Value` 트리에 대한 동시 읽기/쓰기는 외부에서 별도로 동기화해야 합니다. |
