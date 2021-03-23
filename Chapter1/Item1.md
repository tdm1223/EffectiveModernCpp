# 항목 1. 템플릿 형식 연역 규칙을 숙지하라

```cpp
template<typename T>
void f(ParamType param);
f(expr);
```

- 컴파일 도중 컴파일러는 `expr`을 이용해서 두가지 형식을 연역한다.
  1. T
  2. ParamType
- 두 형식이 다른 경우가 많다.

```cpp
template<typename T>
void f(const T& param);

int x = 0;
f(x);
```
- `T`는 `int`로 연력되나 `ParamType`은 `const int&`로 연역된다.
- `T`에 대해 연역된 형식이 함수에 전달된 인수의 형식과 항상 같지는 않다 (`T`의 형식 != `expr`의 형식)
- `T`에 대해 연역된 형식은 `expr`의 형식에 의존할 뿐만 아니라 `ParamType`의 형태에도 의존한다.
  - 형태에 따라 세가지 경우로 나뉜다.

## 경우 1: ParamType이 포인터 또는 참조형식이지만 보편 참조는 아님
### 형식 연역 과정
1. `expr`이 참조 형식이면 참조 부분을 무시한다.
2. `expr`의 형식을 `ParamType`에 대해 패턴 부합 방식으로 대응시켜서 `T`의 형식을 결정한다.

```cpp
template<typename T>
void f(T& param);

int x = 27
const int cx = x;
const int& rx = x;

f(x); // T는 int, param의 형식은 int&
f(cx); // T는 const int, param의 형식은 const int&
f(rx); // T는 const int, param의 형식은 const int&
```
- 두번째, 세번째 `f` 함수 호출에서 `cx`와 `rx`에 `const` 값이 배정되었기 때문에 `T`가 `const int`로 연역된다.
  - 매개변수 `param`의 형식은 `const int&`가 되었다.
- 세번째 `f` 함수 호출에서 `rx`의 형식이 참조지만 `T`는 비참조로 연역된다.
  - 연역 과정에서 `rx`의 참조성이 무시되었기 때문이다.

### f의 매개변수의 형식을 T&에서 const T&로 바꾼다면?

```cpp
template<typename T>
void f(const T& param);

int x = 27
const int cx = x;
const int& rx = x;

f(x);  // T는 int, param의 형식은 const int&
f(cx); // T는 int, param의 형식은 const int&
f(rx); // T는 int, param의 형식은 const int&
```
- `param`이 `const`에 대한 참조로 간주되므로, `const`가 `T`의 일부로 연역될 필요가 없다.
- 형식 연역 과정에서 `rx`의 참조성은 무시된다.

### param이 참조가 아니라 포인터라면?
- `param`이 참조가 아니라 포인터(또는 const를 가리키는 포인터)라도 형식 연역은 같은 방식으로 진행된다.

```cpp
template<typename T>
void f(T* param);

int x = 27;
const int *px = &x;

f(&x);  // T는 int, param의 형식은 int *
f(&px); // T는 const int, param의 형식은 const int*
```

## 경우 2: ParamType이 보편 참조임
- 템플릿이 보편 참조 매개변수를 받는 경우에는 상황이 조금 불투명해 진다.
- 매개변수의 선언은 오른값 참조와 같은 모습이지만 왼값 인수가 전달되면 오른값 참조와는 다른 방식으로 행동한다.
- `expr`이 왼값이면, `T`와 `ParamType` 둘 다 왼값 참조로 연역된다.
  - 템플릿 형식 연역에서 T가 참조 형식으로 연역되는 경우는 이것이 유일하다.
  - `ParamType`의 선언 구문은 오른값 참조와 같은 모습이지만, 연역된 형식은 왼값 참조이다.
- `expr`이 오른값이면 정상적인(경우 1) 규칙들이 적용된다.

```cpp
template<typename T>
void f(T&&) param);

int x = 27
const int cx = x;
const int& rx = x;

f(x);  // x는 왼값. T는 int&, param의 형식은 int&
f(cx); // cx는 왼값. T는 const int&, param의 형식은 const int&
f(rx); // rx는 왼값. T는 const int&, param의 형식은 const int&
f(27); // 27은 오른값. T는 int, param의 형식은 int&&
```
- 보편 참조 매개변수에 관한 형식 연역 규칙들이 왼값 참조나 오른값 참조 매개변수들에 대한 규칙들과는 다르다.
- 이 예들이 각각 해당 형식으로 연역되는 구체적인 이유는 [항목24](/Chapter5/Item24.md)에서 설명한다.

## 경우 3: ParamType이 포인터도 아니고 참조도 아님
- `ParamType`이 포인터도 아니고 참조도 아니라면, 인수가 함수에 값으로 전달되는 상황인 것이다.
- `param`은 주어진 인수의 복사본으로 완전히 새로운 객체이다.
- `param`이 새로운 객체라는 사실 때문에 `expr`에서 `T`가 연역되는 과정에서 아래와 같은 규칙들이 적용된다.
  1. `expr`의 형식이 참조이면, 참조 부분은 무시된다.
  2. `expr`의 참조성을 무시한 후 `expr`이 `const`이면 `const` 역시 무시한다.
  3. [volatile](/Chapter7/Item40.md)일 경우에도 무시한다.

```cpp
template<typename T>
void f(T param);

int x = 27;
const int cx = x;
const int& rx = x;

f(x);  // T와 param의 형식은 둘 다 int
f(cx); // T와 param의 형식은 둘 다 int
f(rx); // T와 param의 형식은 둘 다 int
```
- `cx`와 `rx`가 `const` 값을 칭하지만 `param`은 `const`가 아니다.
  - `param`은 `cx`나 `rx`의 복사본이므로 `param`은 `cx`나 `rx`와는 완전히 독립적인 객체이다.
- 명심할 것은 `const`가 값 전달 매개변수에 대해서만 무시된다는 점이다.

## 배열 인수
- 배열과 포인터를 맞바꿔 쓸 수 있는 것처럼 보이는 환상의 주된 원인은, 많은 문맥에서 배열이 배열의 첫 원소를 가리키는 포인터로 붕괴한다는 점이다.

```cpp
const char name[] = "J. P. Briggs"; // name의 형식은 const char[13]
const char* ptrToName = name; // 배열이 포인터로 붕괴된다.

template<typename T>
void f(T param); // 값 전달 매개변수가 있는 템플릿
f(name); // T와 param에 대해 연역되는 형식들은?
```

- 배열 매개변수 선언이 포인터 매개변수처럼 취급되므로, 템플릿 함수에 값으로 전달되는 배열의 형식은 포인터 형식으로 연역된다.
- 위 예에서 형식 매개변수 `T`는 `const char*`로 연역된다. (배열이지만 `const char*`로)

### 배열에 대한 참조로 선언한다면?

```cpp
template<typename T>
void f(T& param);
f(name);
```

- `T`에 연역된 형식은 배열의 실제 형식이 된다.
- 형식은 배열의 크기를 포함하므로 이 예에서 `T`는 `const char[13]`으로 연역되고 f의 매개변수의 형식은 `const char(&)[13]`으로 연역된다.

### 배열에 담긴 원소들의 개수를 연역하는 템플릿

```cpp
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept
{
  return N;
}

int keyVals[] = {1, 2, 3, 4, 5};
int mappedVals[arraySize(keyVals)]; // 원소 개수가 5개인 새 배열 선언 가능
```

- [함수를 constexpr로 선언하면 함수 호출의 결과를 컴파일 도중에 사용](/Chapter3/Item15.md)할 수 있게 된다.
- 위 템플릿을 사용하면 중괄호 초기화 구문으로 정의된 기존 배열과 같은 크기의 새 배열을 선언하는 것이 가능해진다.

## 함수 인수
- 함수 형식도 함수 포인터로 붕괴할 수 있다.
- 배열에 대한 형식 연역과 관련된 내용은 모두 함수에 대한 형식 영역에, 함수포인터로의 붕괴에 적용된다.

```cpp
void func(int, double); // 형식이 void(int, double)인 함수

template<typename T>
void f1(T param); // 값 전달 방식
f1(func); // param은 함수 포인터로 연역된다. 형식은 void(*)(int, double)

template<typename T>
void f2(T& param); // 참조 전달 방식
f2(func); // param은 함수 참조로 연역된다. 형식은 void(&)(int, double)
```
