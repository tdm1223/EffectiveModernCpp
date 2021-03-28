# 항목 3. decltype의 작동 방식을 숙지하라
- `decltype`은 주어진 이름이나 표현식의 형식을 알려준다.
- 대부분의 경우 예측한 대로 형식을 말해준다. (아주 가끔 예상 밖의 결과를 제공한다)
- `C++11`에서 `decltype`은 함수의 반환 형식이 그 매개변수 형식들에 의존하는 함수 템플릿을 선언할 때 주로 쓰인다.

## 예측 가능한 형식
```cpp
const int i = 0; // decltype(i)는 const int

bool f(const Widget& w); // decltype(w)는 const Widget&, decltype(f)는 bool(const Widget&)

struct Point{
  int x, y;
}; // decltype(Point::x)는 int

Widget w; // decltype(w)는 Widget

if (f(w)) // decltype(f(w))는 bool

if (v[0] == 0) // decltype(v[0])는 int&
```

## 특이한 형식
```cpp
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i) -> decltype(c[i])
{
    authenticateUser();
    return c[i];
}
```
- `auto` 와 `decltype` 을 이용한 **후행 반환 형식**이다.
  - 후행 반환 형식으로 선언한 함수의 반환 타입은 `decltype` 에 주어진 객체의 타입이 된다.
  - 여기서는 `c[i]` 의 타입이다.
- 만약 `c[i]` 의 반환 타입이 `T&` 인 경우라면 생각한대로 동작하지 않는다.
  - [항목 1의 형식 연역 규칙 3](/Chapter1/Item1.md)에 의해 참조성이 사라지기 때문이다. 

### 형식 연역을 위한 decltype(auto)
- `decltype` 과 `auto` 를 합쳐서 해결할 수 있다.
- `decltype(auto)`는 `auto` 로 형식을 연역하는데 그 과정에서 `decltype`의 규칙을 사용하겠다는 의미이다.

### 문제를 수정한 코드
```cpp
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container& c, Index i)
{
    authenticateUser();
    return c[i];
}
```

## 두가지 문제점
### 1. 우측값 참조를 위한 보편참조
- c는 우측값으로 넘어 올 수 없다.
- `c[i]` 가 참조인 경우 `Container` 는 `const` 로 지정될 수 없기 때문이다.
- 보편참조를 사용하여 `Container& c` 파라미터를 `Container&& c` 로 바꾸면 된다.

```cpp
template<typename Container, typename Index>
decltype(auto)
authAndAccess(Container&& c, Index i)
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

### 2. 두번째 특이한 예외 상황
- 아주 가끔 예상 밖의 결과를 제공하는 상황이다
```cpp
decltype(auto) f1()
{
  int x = 0;
  ...
  return x;
}

decltype(auto) f2()
{
  int x = 0;
  ...
  return (x);
}
```
- `f1`과 `f2`는 똑같은 함수처럼 보이지만 반환 타입은 다르다.
- 첫번째는 `int` 타입이지만 두번째는 `int&` 가 된다.
  - 이름을 괄호로 감싸면 `decltype`이 보고하는 형식이 달라진다!
  - `C++11`에서는 드물게 만나는 현상이지만 `decltype(auto)`를 지원하는 `C++14`에서는 `return` 문 작성 습관의 사소한 차이 때문에 함수의 반환 형식 연역 결과가 달라지는 사태가 벌어질 수 있다.