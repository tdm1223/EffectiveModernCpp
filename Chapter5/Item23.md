# 항목 23. std::move와 std::forward를 숙지하라
- `std::move`는 모든 것을 **이동**하지는 않는다.
- `std::forward`는 모든 것을 **전달**하지는 않는다.
- 실행시점에서는 둘 다 아무것도 하지 않는다.
  - 실행 가능 코드를 단 한 바이트도 산출하지 않는다.
- `std::move`와 `std::forward`는 캐스팅을 수행하는 **함수 템플릿**이다.

## std::move
- 주어진 인수를 무조건 **오른값으로 캐스팅**한다.
- 실제로 객체를 이동시켜주는 것이 아니고 **오른값 참조로 캐스팅**을 해줄 뿐이다.
  - 이동될 수 있는 조건만 갖추게 해주는 함수이다.
  - 실제로 이동을 시키는 것도 이동을 보장하는 것도 아니다.
- 오른값 참조로 인자를 넘겼다고 해서 무조건 이동이 되는게 아니다.

### C++ 11 스타일
```cpp
template<typename T>
typename remove_reference<T>::type&&
move(T&& param)
{
  using ReturnType = typename remove_reference<T>::type&&; // 별칭 선언
  return static_cast<ReturnType>(param);
}
```
- 객체에 대한 참조(정확히는 [보편 참조](/Chapter5/Item24.md))를 받아서 같은 객체에 대한 오른값 참조를 돌려준다.
- [형식 T가 왼값 참조이면 T&&는 왼값 참조가 된다.](/Chapter5/Item28.md)
  - 이를 방지하기 위해 T에 [형식 특질](/Chapter3/Item9.md) `std::remove_reference`를 적용한다.
  - 반환 형식의 &&는 항상 **참조가 아닌 형식에 적용**되고 `std::move`는 반드시 오른값 참조를 돌려준다.

### C++14 스타일
- `C++14`에서는 [함수 반환 형식 연역](/Chapter1/Item3.md)과 표준 라이브러리의 별칭 템플릿들 중 하나인 [std::remove_reference_t](/Chapter3/Item9.md)를 활용해 간결하게 작성할 수 있다.
```cpp
template<typename T>
decltype(auto) move(T&& param)
{
  using ReturnType = remove_reference_t<T>&&;
  return static_cast<ReturnType>(param);
}
```

### std::move와 const
- 이동 함수가 캐스팅만 해주다보니 `std::move` 함수를 사용하더라도 이동이 되지 않는 경우가 있다.
- 이동 연산을 허용하고 싶다면 객체의 타입으로 `const`를 지정해선 안된다.
```cpp
class Annotation {
public:
  explicit Annotation(const std::string text)
    : value(std::move(text)) // text를 value로 이동하려고 한다
  { ... }
  ...
    
private:
  std::string value;
};
```
- `Annotation` 클래스는 멤버 변수 `value`를 `text`의 내용으로 설정하는 클래스이다.
- `std::move`를 이용해 `text`를 `value`로 **이동**시키려고 했지만 실제로는 **복사**가 된다.

```cpp
// std::string의 일부
class string
{
public:
    ...
    string(const string& rhs);  // 복사 생성자
    string(string&& rhs);       // 이동 생성자
    ...
};
```
- `Annotation`에서 멤버를 초기화할 때 `text`는 생성자에서 `const std::string&&` 타입으로 캐스팅 된다.
- 이동 생성자는 `std::string&&`을 타입으로 받고 `const`의 차이 때문에 이동 생성자는 호출될 수가 없다.
- 복사 생성자가 인자로 받는 `const string&`과 `const string&&`은 서로 호환이 되기 때문에 **복사 생성자가 호출**된다. 
- 프로그램의 동작 자체는 이동과 복사가 같기 때문에 실제로 이동 생성자 호출되는지 복사 생성자가 호출되는지 알기 힘들다.
- 이동하지 않을 뿐만 아니라 캐스팅되는 객체가 **이동 자격을 갖추게 된다는 보장**도 제공하지 않는다는 것을 알 수 있다.

## std::forward
- `std::forward`와 `std::move`는 거의 같은 역할을 한다.
- `std::foward`는 주어진 인수가 **오른값에 묶인 경우에 오른값으로 캐스팅**한다.

### std::forward 용법
- **보편 참조 매개변수**를 받아 **다른 함수에 전달**할때 사용한다.
```cpp
void process(const Widget& lvalArg);                // 왼값들을 처리하는 함수
void process(Widget&& rvalArg);                     // 오른값들을 처리하는 함수

template<typename T>                                // param을 process에
void logAndProcess(T&& param)                       // 넘겨주는 템플릿
{
  auto now = std::chrono::system_clock::now();    // 현재 시간을 얻음
    
  makeLogEntry("Calling 'process'", now);
  process(std::forward<T>(param));
}

Widget w;

logAndProcess(w);                                   // 왼값으로 호출
logAndProcess(std::move(w));                        // 오른값으로 호출
```
- `logAndProcess`는 `param`을 `process`함수에 전달한다.
- `process`는 왼값과 오른값에 대해 **오버로딩** 되어 있다.
  - 왼값으로 호출하면 왼값을 받는 `process`로 전달된다.
  - 오른값으로 호출하면 오른값을 받는 `process`로 전달된다.
- `param`은 하나의 왼값이다.
  - `logAndProcess` 내부에서 일어나는 모든 `process` 호출은 결국 `process`의 **왼값 오버로딩 버전**을 실행하게 된다.
- 이를 방지하기 위해 `param`을 초기화하는데 쓰인 인수가 **오른값일때만 param을 오른값으로 캐스팅**하는 메커니즘이 필요하다.
  - 이럴때 사용하는것이 `std::forward`이다.
  - `std::forward`는 인자가 **오른값으로 초기화되었을 때에만 오른값으로 캐스팅**해준다.
- `std::forward`는 인자로 넘어온 값이 오른값으로 초기화되었는지 아닌지를 판단하기 위해 템플릿 타입 인자 T를 활용한다.
  - T 타입 안에 인자로 넘어온 값이 오른값인지 아닌지 판단할 수 있는 정보가 들어있고 `std::forward`는 그걸 이용해서 판단한다.
  - [정보가 logAndProcess의 템플릿 매개변수 T에 부호화 되어있다.](/Chapter5/Item28.md)

## std::move와 std::forward 비교
- 둘 다 오른값으로 캐스팅하는게 목적이고 `std::forward`는 **타입 인자로 값 타입이 넘어오면 오른값으로 캐스팅** 한다.
- `std::forward`만 사용해도 되지 않을까 ?
  - 기술적인 관점에서는 가능하다. (`std::forward`로 모든 것을 할 수 있다.)
  - 두 가지를 분리해서 쓰는게 깔끔하고 가독성도 좋다.

```cpp
class Widget {
public:
  Widget(Widget&& rhs) : s(std::move(rhs.s))
  {
    ++moveCtorCalls;
  }
private:
  static std::size_t moveCtorCalls;
  std::string s;
};

class Widget {
public:
  Widget(Widget&& rhs) : s(std::forward<std::string>(rhs.s)) // 관례에서 벗어난 바람직하지 않은 구현
  {
    ++moveCtorCalls;
  }
private:
  static std::size_t moveCtorCalls;
  std::string s;
};
```
- `std::move`에서는 **함수 인수**(`rhs.s`)만 지정하면 된다.
- `std::forward`에서는 **함수 인수**(`rhs.s`)와 **템플릿 형식 인수**(`std::string`) 둘다 지정해야 한다.
  - [전달되는 인수가 오른값임을 부호화하는 데 쓰이는 관례이다.](/Chapter5/Item28.md)