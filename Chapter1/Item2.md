# 항목 2. auto의 형식 연역 규칙을 숙지하라
- 한 가지 기이한 예외를 빼면 `auto` 형식 연역이 곧 템플릿 형식 연역이다.
- [항목1](/Chapter1/Item1.md)에서처럼 `auto` 형식 연역 역시 세 가지로 나뉘고 잘 동작한다.

## 템플릿 형식 연역과 다른점
### C++에서의 변수 초기화 방식
```cpp
// C++98에서 초기화
int x1 = 27;
int x2(27);

// 균일 초기화를 지원하는 C++11에서 추가된 초기화
int x = { 27 };
int x{ 27 };
```

### auto를 통한 초기화
```cpp
auto x = 27;
auto x(27);

auto x = { 27 };
auto x{ 27 };
```
- 아래 두가지 초기화 방식은 값이 27인 원소 하나를 담은 `std::initializer_list<int>`형식의 변수를 선언한다.
- `auto`로 선언된 변수의 초기치가 **중괄호** 쌍으로 감싸인 형태면 연역된 형식은 `std::initializer_list`이다.
  - 그런 형식을 연역할 수 없으면 컴파일이 거부된다. `auto x = {1, 2, 3.0};`(오류)

```cpp
auto x = {11, 23, 9}; // x의 형식은 std::initializer_list<int>

template<typename T>
void f(T param);

f({11, 23, 9}); // 오류! T에 대한 형식을 연역할 수 없음

template<typename T>
void f(std::initializer_list<t> initList_);

f({11, 23, 9}); // T는 int로 연역되며, initList의 형식은 std::initializer_list<int>로 연역된다.
```
- `auto` 형식 연역과 템플릿 형식 연역의 실질적인 차이는 `auto`는 중괄호 초기치가 `std::initilizer_list`를 나타낸다고 가정하지만 템플릿 형식 연역은 그렇지 않다는 것이다.

## C++14에서 auto 반환 함수
- `C++14`에서는 함수의 반환 형식을 `auto`로 지정해서 [컴파일러가 연역](/Chapter1/Item3.md)하게 만들 수 있고 람다의 매개변수 선언에 `auto`를 사용하는것도 가능하다.
- `auto`의 용법들에는 **auto 형식 연역**이 아니라 **템플릿 형식 연역**의 규칙들이 적용된다.
  - 중괄호 초기치를 돌려주는 함수의 반환 형식을 `auto`로 지정하면 컴파일이 실패한다.

```cpp
auto createInitList()
{
  return { 1, 2, 3}; // {1, 2, 3}의 형식을 연역할 수 없기에 오류
}
```

- C++14 람다의 매개변수 형식 명세에 `auto`를 사용하는 경우에도 마찬가지 이유로 컴파일이 실패한다.

```cpp
std::vector<int> v;
auto resetV = [&v](const auto& newValue) { v = newValue; };
resetV({1, 2, 3}); // {1, 2, 3}의 형식을 연역할 수 없기에 오류
```
