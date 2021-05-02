# 항목 28. 참조 축약을 숙지하라
## 참조 축약
```cpp
template<typename T>
void func(T&& param);
```
- [템플릿 매개변수에 대해 연역된 형식에는 전달된 인수가 왼값이었는지 오른값이었는지에 대한 정보가 부호화된다.](/Chapter5/Item23.md)
  - 인자로 **왼값**이 넘어오면 `T`는 **왼값 참조**로 연역된다.
  - 인자로 **오른값**이 넘어오면 `T`는 **비참조 형식**으로 연역된다.
```cpp
Widget widgetFactory();   // 오른값을 돌려주는 함수
Widget w;                 // 변수 (왼값)
func(w);                  // func를 왼값으로 호출한다 (T는 Widget&로 연역된다)
func(widgetFactory());    // func를 오른값으로 호출한다 (T는 Widget으로 연역된다)
```
- 보편 참조를 사용하는 `func` 함수는 **매개변수**에 따라 연역되는 형식이 달라진다.

### 보편 참조를 받는 함수 템플릿에 왼값을 넘겨줄 경우
```cpp
template<typename T>
void func(T&& param);

Widget w;
func(w);    
```
- T에 대해 연역된 현식(`Widget&`)으로 템플릿을 인스턴스화한 결과는 아래와 같을 것이다.
```cpp
void func(Widget& && param);
```
- **참조에 대한 참조**가 발생한다. `C++`에서는 참조에 대한 참조는 위법이다.
- [보편 참조 param은 왼값으로 초기화되므로 param의 형식은 왼값 참조가 된다.](/Chapter5/Item24.md)
- 컴파일러가 `T`에 대해 연역된 형식을 템플릿에 대입하면 참조에 대한 참조처럼 서명이 나오지만 실제로 만들어지는 함수 서명은 아래와 같다.
```cpp
void func(Widget& param);
```
- `Widget& &&`에서 `Widget&`로 변경되는 이것이 **참조 축약**이다.
- 참조에 대한 참조가 위법인 것은 사실이지만 **특정 문맥에서는 컴파일러가 참조에 대한 참조를 산출하는 것이 허용**된다.

### 참조에 대한 참조
- 참조는 **두 종류이**므로 참조에 대한 참조로 가능한 조합은 총 **네가지**이다.
  - <span style="color:red">왼값</span>에 대한 <span style="color:red">왼값</span>
  - <span style="color:blue">오른값</span>에 대한 <span style="color:red">왼값</span>
  - <span style="color:red">왼값</span>에 대한 <span style="color:blue">오른값</span>
  - <span style="color:blue">오른값</span>에 대한 <span style="color:blue">오른값</span>
- 참조에 대한 참조가 허용되는 문맥에서 참조에 대한 참조는 아래 규칙에 따라 하나의 참조로 축약된다.
  - **두 참조 중 하나**라도 **왼값 참조**이면 결과는 **왼값 참조**이다.
  - **둘다 오른값 참조**일 떄만 결과는 **오른값 참조**이다.
- `std::forward`가 작동하는 것도 참조 축약 덕분이다.

## std::forward의 원리
- `std::forward`의 간략한 구현이다.
```cpp
template<typename T>
T&& forward(typename remove_reference<T>::type& param)
{
  return static_cast<T&&>(param);
}
```
### 왼값이 전달되었을 경우
- `f`에 전달된 인수가 `Widget` 형식의 **왼값**이라 가정한다.
- `T`는 `Widget&`로 연역되며 `std::forward` 호출은 `std::forward<Widget&>` 형태로 인스턴스화 된다.
- `Widget&`를 `std::forward`의 구현에 대입하면 아래와 같은 코드가 나온다.
```cpp
Widget& && forward(typename remove_reference<Widget&>::type& param)
{
  return static_cast<Widget& &&>(param);
}
```
- [형식 특질 std::remove_reference<Widget&>::type은 Widget을 산출](/Chapter3/Item9.md)하므로 아래처럼 바꿀 수 있다.
```cpp
Widget& && forward(Widget& param)
{
    return static_cast<Widget& &&>(param);
}
```
- 참조 축약은 **반환 형식**과 **캐스팅**에도 적용된다.
- 참조 축약이 적용된 `std::forward`의 최종 버전은 아래와 같다.
```cpp
Widget& forward(Widget& param)
{
  return static_cast<Widget&>(param);
}
```
- 함수 템플릿에 **왼값 인수**가 전달되면 `std::forward`는 **왼값 참조**를 받아서 **왼값 참조**를 돌려주는 형태로 인스턴스화 된다.
### 오른값이 전달되었을 경우
```cpp
Widget&& forward(typename remove_reference<Widget>::type& param)
{
  return static_cast<Widget&&>(param);
}
```
- `std::remove_reference`를 비참조 형식 `Widget`에 적용하면 원래의 형식인 `Widget`이 산출되므로 아래처럼 바꿀 수 있다.
```cpp
Widget&& forward(Widget& param)
{
  return static_cast<Widget&&>(param);
}
```
- 참조에 대한 참조가 없으므로 참조 축약도 없다.
- 함수가 돌려준 **오른값 참조**는 **오른값**이므로 `std::forward`는 `f`의 매개변수 `fParam`(왼값)을 **오른값**으로 바꾼다.
- `f`에 전달된 오른값 인수는 하나의 **오른값**으로서 `somfFunc`에 전달되며 이것은 마땅히 그래야 하는 결과이다.

### C++14에서 std::forward
- `C++14`에서는 `std::reference_t` 덕분에 더 간결하게 구현할 수 있다.
```cpp
template<typename T>
T&& forward(remove_reference_t<T>& param)
{
    return static_cast<T&&>(param);
}
```

## 참조 축약이 일어나는 문맥
- 참조 축약이 일어나는 문맥은 네 가지이다.

### 1. 템플릿 인스턴스화
- **템플릿을 인스턴스화**하는 과정에서 **참조에 대한 참조**가 나타나면 **참조 축약**을 적용시킨다.
- 위에서 본 예제와 같다.

### 2. auto 변수에 대한 형식 연역
- 세부사항은 템플릿 인스턴스화의 경우와 같다.
- [auto 변수의 형식 연역은 템플릿의 형식 연역과 본질적으로 같기 때문](/Chapter1/Item2.md)이다.

```cpp
Widget widgetFactory();   // 오른값을 돌려주는 함수
Widget w;                 // 변수(왼값)
func(w);                  // func를 왼값으로 호출한다. T는 Widget&로 연역된다.
func(widgetFactory());    // func를 오른값으로 호출한다. T는 Widget으로 연역된다.
```

#### 참조 축약이 일어나는 경우
```cpp
auto&& w1 = w;
Widget& && w1 = w;
Widget& w1 = w;     // 참조 축약
```

- `w1`을 왼값으로 초기화 하며 `auto`의 형식은 `Widget&`로 연역된다.
- `Widget&`를 `w1` 선언의 `auto`에 대입하면 참조 대 참조가 나타나고 참조 축약이 일어나게 된다.
- 결과적으로 `w1`은 왼값 참조이다.

#### 참조 축약이 일어나지 않는 경우

```cpp
auto&& w2 = widgetFactory();
Widget&& w2 = widgetFactory();
```
- `w2`를 오른값으로 초기화 하며 `auto`는 비참조 형식 `Widget`으로 연역된다.
- `Widget`을 `auto`에 대입하면 참조 대 참조가 없으므로 이것이 끝이다.
- `w2`는 오른값 참조이다.


#### 보편 참조 결론
- 아래 두 조건이 만족되는 문맥에서 **보편 참조**는 사실상 **오른값 참조**이다.
  - 형식 연역에서 왼값과 오른값이 구분된다. `T`형식의 왼값은 형식 `T&`로 연역되고, `T`형식의 오른값은 형식 `T`로 연역된다.
  - 참조 축약이 적용된다.

### 3. typedef와 별칭 선언의 지정 및 사용
- `typedef`가 지정 또는 평가되는 도중에 참조에 대한 참조가 발생하면 참조 축약이 끼어들어서 참조에 대한 참조를 제거한다.
```cpp
template<typename T>
class Widget {
public:
    typedef T&& RvalueRefToT;
    ...
};

Widget<int&> w; // 왼값 참조 형식 인스턴스화
```
- `Widget`을 왼값 참조 형식으로 인스턴스화 하였다.
- `Widget`의 `T`를 `int&`로 대체하면 `typedef`는 아래와 같이 된다.

```cpp
typedef int& && RvalueRefToT;
typedef int& RvalueRefToT;
```
- **참조 축약**이 발생한다.
- `Widget`을 **왼값 참조 형식**으로 인스턴스화하면 `RvalueRefToT`는 오른값 참조가 아니라 **왼값 참조**에 대한 `typedef`가 된다.

### 4. decltype 사용
- 컴파일러가` decltype`에 관여하는 형식을 분석하는 도중에 **참조에 대한 참조**가 발생하면 **참조 축약**이 발생한다.

```cpp
int& func(int k);
decltype(func(3))& t;
```
- `decltype(func(3))`이 `int&`로 바뀐다.
- 참조 축약이 발생하여 `int& & t`가 `int& t`로 바뀐다.