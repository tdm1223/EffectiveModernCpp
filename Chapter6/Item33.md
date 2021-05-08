# 항목 33. std::forward를 통해서 전달할 auto&& 매개변수에는 decltype을 사용하라
## 일반적 람다
- `C++14`에서 가장 고무적인 기능은 **일반적 람다**로 매개변수에 `auto`를 사용하는 람다이다.
- 구현은 간단한데, 람다의 클로저 클래스의 `operator()`를 템플릿 함수로 만들면 된다.
```cpp
auto f = [](auto x) { return normalize(x); };
```
- 이 람다가 산출하는 **클로저 클래스**의 함수호출 연산자는 아래와 같은 모습이다.
```cpp
class 컴파일러가_생성한_클래스_이름 {
public:
  template<typename T>
  auto operator()(T x) const
  { return normalize(x); }
  ...
};
```
- 람다는 매개변수 `x`를 그냥 `nomalize`로 전달하기만 한다.
- `normalize`가 **왼값**과 **오른값**을 다른 방식으로 처리한다면 이 람다는 제대로 작성한 것이 아니다.
  - 이 람다는 `normalize`에 항상 **왼값**(매개변수 x)을 전달하기 때문이다.
- 자신에게 주어진 인수가 **오른값**이라면 왼값이 아니라 **오른값**을 전달해야 마땅하다.
  - 코드를 고치는 것은 자명하다.
- `normalize`에 완벽하게 전달하려면 두 가지를 바꿔야 한다.
  - `x`가 [보편 참조](/Chapter5/Item24.md)여야 한다.
  - `x`를 [std::forward](/Chapter5/Item25.md)를 통해서 `normalize`에 전달해야 한다.

### 수정한 코드
```cpp
auto f = [](auto&& x)
         { return normalize(std::forward<???>(x)); };
```
- ???에 적어야할 타입은 무엇일까?
  - 보통의 경우 완벽 전달은 형식 매개변수 `T`를 받는 템플릿 함수 안에서 사용하고 `std::forward<T>`라고 하면 그만이다.
  - 람다에는 그런식으로 사용할 형식 매개변수 `T`가 없다.
  - 람다가 산출하는 클로저 클래스의 템플릿 `operator()`에는 `T`가 있지만 람다에서 그 T를 지칭할수 없으므로 무용지물이다.
- 람다에 주어진 인수가 왼값인지 오른값인지는 매개변수 `x`의 형식을 조사하면 알 수 있다.
  - 이런 조사에 사용하는 수단이 [decltype](/Chapter1/Item3.md)이다.
  - **왼값**이 전달되었다면 `decltype(x)`는 **왼값 참조**에 해당하는 형식을 산출한다.
  - **오른값**이 전달되었다면 `decltype(x)`는 **오른값 참조** 형식을 산출한다.

### std::forward와 decltype
- `std::forward` 호출 시 전달할 인자가 왼값임을 나타내기 위해서는 **왼값 참조 형식 인자**를 사용한다.
- `std::forward` 호출 시 전달할 인자가 오른값임을 나타내기 위해서는 **비참조 형식 인자**를 사용한다.
  - `x`가 오른값에 묶인다면 `decltype(x)`는 **오른값 참조를 산출**하는데 이는 관례(비참조)와 맞지 않다.
- 아래는 `std::forward`의 `C++14` 구현이다.
```cpp
template<typename T>
T&& forward<remove_reference_t<T>& param)
{
    return static_cast<T&&>(param);
}
```
- 클라이언트 코드가 `Widget` 형식의 오른값을 완벽하게 전달할 때에는 `Widget` 형식으로 `std::forward`를 인스턴스화 할 것이다.
```cpp
Widget&& forward(Widget& param)
{
    return static_cast<Widget&&>(param);
}
```
- 클라이언트가 `Widget` 형식의 동일한 오른값을 완벽 전달하되 `T`를 **비참조 형식**으로 지정하는 관례를 따르지 않고 오른값 참조 형식으로 지정하면 아래처럼 될것이다.
```cpp
Widget&& && forward(Widget& param)
{
    return static_cast<Widget&& &&>(param);
}
```
- 오른값 참조에 대한 오른값 참조는 단일한 오른값 참조가 된다는 **참조 축약 규칙**을 적용하면 아래와 같은 모습이 된다.
```cpp
Widget&& forward(Widget& param)
{
    return static_cast<Widget&&>(param);
}
```
- 이 인스턴스를 `T`를 `Widget`으로 지정해서 `std::forward`를 호출했을 때의 인스턴스(`std::forward<Widget>`)와 비교하면 같다는 것을 알 수 있다.
- `decltype`이 산출하는 형식이 관례와 맞지 않아도, 관례적 형식을 사용했을 떄와 같은 결과가 나온다.
- 왼값이든 오른값이든 `decltype(x)`를 `std::forward`로 넘겨주면 원하는 결과가 된다.
- **완벽 전달 람다**는 아래와 같이 작성하면 된다.
- 마침표 세 개를 두 번 추가하면 임의의 개수의 매개변수들을 받아서 완벽하게 전달하는 람다를 작성할 수 있다.
```cpp
auto f = [](auto&& x)
         {
           return func(normalize(std::forward<decltype(x)>(x)));
         };

auto f = [](auto&&... xs)
         {
           return func(normalize(std::forward<decltype(xs)>(xs)...));
         };
```