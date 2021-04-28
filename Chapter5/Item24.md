# 항목 24. 보편 참조와 오른값 참조를 구별하라
## 오른값 참조, 보편 참조
- 어떤 타입 T에 대한 오른값 참조는 T&&로 나타낸다.
- 소스 코드에 나타나는 T&&들은 모두 타입 T에 대한 오른값 참조로 봐도 될 거라 생각 할 수 있지만 그렇지 않다.

```cpp
void f(Widget&& param);             // 오른값 참조

Widget&& var1 = Widget();           // 오른값 참조

auto&& var2 = var1;                 // 오른값 참조 아님

template<typename T>
void f (std::vector<T>&& param);    // 오른값 참조

template<typename T>
void f(T&& param);                  // 오른값 참조 아님
```

### T&&의 두가지 의미
1. 오른값 참조
   - 오른값 참조는 오른값에만 묶이며 일차적인 존재의 이유는 **이동의 원본이 될 수 있는 객체를 지정**하는 것이다.
2. 해당 타입이 오른값 참조 **또는** 왼값 참조 **중 하나** (보편 참조)
   - 소스 코드에서는 **오른값 참조(T&&)**처럼 보이지만 때에 따라서는 **왼값 참조(T&)**인것처럼 행동한다.
   - 오른값에 묶을 수도 있고 왼값에 묶을 수도 있다.
   - `const` 객체에 묶을 수도 있고 `비const`객체에 묶을 수도 있다.
   - `volatile` 객체에 묶을 수도 있고 `비volatile` 객체에 묶을 수도 있다.
   - `const`이자 `volatile`인 객체에도 묶을 수 있다.

## 보편 참조가 나타나는 두 가지 경우
- 두 가지 모두 **형식 연역이 일어난다는 공통점**이 있다.
- 형식 연역이 일어나지 않는 문맥에서 `T&&`를 사용하면 **오른값 참조**이다.

### 함수 템플릿 매개 변수
```cpp
template<typename T>
void f(T&& param); // param은 보편 참조
```
- `param`의 형식이 연역된다.

### auto 선언
```cpp
auto&& var2 = var1;
```
- `var2`의 형식이 연역된다.

### 보편 참조의 초기치
- 보편 참조는 참조이므로 **반드시 초기화**해야 한다.
- 보편 참조가 오른값 참조를 나타내는지 왼값 참조를 나타내는지는 **보편 참조의 초기치**가 결정한다.
- 초기치가 오른값이면 보편 참조는 오른값 참조에 해당한다.
- 초기치가 왼값이면 보편 참조는 왼값 참조에 해당한다.
- 보편 참조가 **함수의 매개변수**인 경우 초기치는 그 함수를 **호출하는 시점**에서 제공한다.

```cpp
template<typename T>
void f(T&& param);      // param은 보편 참조

Widget w;
f(w);                   // f에 왼값이 전달됨. param의 형식은 Widget&(왼값 참조)

f(std::move(w))         // f에 오른값이 전달됨. param의 형식은 Widget&&(오른값 참조)
```

- 보편 참조가 나타나기 위해 형식 연역이 필요한 것은 맞지만 그걸로 충분하진 않다.
- 참조 선언의 형태도 정확해야 하는데, 보편 참조가 나타나기 위해서는 반드시 **레퍼런스의 형태가 T&&**이어야 한다.

## 형식 연역이 일어나도 보편 참조가 아닌 경우
- 형식 연역이 필요하지만 필요조건일 뿐 충분조건은 아니라고 하였다.
- 아래처럼 형식 연역이 일어나도 보편 참조가 아닌 경우가 존재하기 때문이다.

### 직접적인 형식 연역이 아닌 경우
```cpp
template<typename T>
void f(std::vector<T>&& param); // param은 오른값 참조

std::vector<int> v;
f(v);                           // 오류! 왼값을 오른값 참조에 묶을 수 없음
```
- `f`호출 시 형식 `T`가 연역된다. (호출자가 `T`를 명시적으로 지정한 경우는 예외)
- `param`의 타입이 `T&&`가 아니라 `std::vector<T>&&`이기 때문에 보편 참조가 될 수 없다.
- `param`의 타입은 무조건 오른값 참조가 된다.

### const를 사용하는 경우
```cpp
template<typename T>
void f(const T&& param);   // 오른값 참조
```
- `const` 한정사만 붙여도 참조는 보편 참조가 되지 못한다.

## 템플릿 안에서 형식 연역이 일어나지 않는 경우
- `T&&`이지만 보편참조가 아닐 경우도 존재한다.
- 템플릿 안에서 형식이 `T&&`인 함수 매개변수를 발견했을 때 반드시 **보편 참조**라고 확신할 수는 없다.
  - 템플릿 안에서는 형식 연역이 반드시 일어난다는 보장이 없기 때문이다.
- `std::vector`의 두 함수 `push_back`과 `emplace_back`에서 차이를 알 수 있다.

### push_back
```cpp
// std::vector 구현의 일부
template<class T, class Allocator = allocator<T>>
class vector {
public:
    ...
    void push_back(T&& x);
    ...
};
```
- `push_back`의 매개변수는 보편 참조가 요구하는 형태이지만 형식 연역이 전혀 일어나지 않는다.
- `push_back`은 반드시 구체적으로 인스턴스화 된 `vector`의 일부여야 하며, 그 인스턴스의 형식은 `push_back`의 선언을 완전하게 결정하기 때문이다.

```cpp
std::vector<Widget> v;

class vector<Widget, allocator<Widget>> {
public:
    void push_back(Widget&& x); // 오른값 참조
    ...
};
```
- `std::vector<Widget> v`라는 선언에 의해 `std::vector`템플릿은 위 코드처럼 구체화 된다.
- `std::vector`가 구체화될 때 `push_back`의 인자도 정해지기 때문에 `push_back`을 호출할 때는 **형식 연역이 일어나지 않는다.**

### emplace_back
- `emplace_back` 멤버 함수는 **형식 연역**을 사용한다.
```cpp
template<class T, class Allocator = allocator<T>>
class vector {
public:
    template<class... Args>
    void emplace_back(Args&&... args);
    ...
};
```
- `emplace_back`이 받는 `Args`의 경우 타입 `T`와는 무관하다.
- `Args`의 경우 형식 연역이 일어나고 이 형태는 `T&&` 형태이므로 보편 참조가 된다.

## auto&&
- `auto&&`의 경우도 형식 연역이 일어나며 `T&&` 형태이므로 보편 참조가 된다.
- `C++14`에서 람다의 매개변수로 `auto&&`를 쓸 수 있기 때문에 아래처럼 유용하게 쓸 수 있다.
```cpp
auto timeFuncInvocation =
    [](auto&& func, auto&&... params)
    {
        beginTimer();
        std::forward<decltype(func)>(func)(             // params로
            std::forward<decltype(params)>(params)...   // func 호출
            );
        // 타이머를 정지하고 경과 시간을 기록한다.
    }
```
- `func`는 어떤 호출 가능 객체와도 묶일 수 있는 보편 참조이다.
- `params`는 임의의 형식, 임의의 개수의 객체들과 묶일 수 있는 0개 이상의 보편 참조들이다.
- `auto` 보편 참조 덕분에 `timeFuncInvocation`함수는 거의 모든 함수의 실행 시간을 측정할 수 있다.