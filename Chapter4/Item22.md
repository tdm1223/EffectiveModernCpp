# 항목 22. Pimpl 관용구를 사용할 때에는 특수 멤버 함수들을 구현 파일에서 정의하라
### Pimpl(pointer to implementation) 관용구
- 클래스의 멤버들을 구현 클래스(또는 구조체)를 가리키는 포인터로 대체한다.
- 일차 클래스에 멤버들을 구현 클래스로 옮기고 포인터를 통해서 자료 멤버들에 간접적으로 접근하는 기법이다.

### 기존 코드
```cpp
// Widget.h 헤더 파일 안
#include<vector>
#include<string>
#include"Gadget.h"

class Widget {
public:
    Widget();
    ...
private:
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;          // Gadget은 사용자 정의 형식
};
```
- `Widget`을 **컴파일**하려면 `#include`를 이용해서 `string`, `vector`, `gadget`를 포함해야 한다.
  - 헤더들 때문에 `Widget` 클라이언트의 컴파일 시간이 증가한다.
- 클라이언트가 헤더들의 내용에 의존하게 된다.
  - 헤더의 내용이 변하면 `Widget` 클라이언트도 **반드시 다시 컴파일** 해야 한다.

### C++98 Pimpl 적용 코드
```cpp
class Widget {
public:
    Widget();
    ~Widget();
    ...
private:
    struct Impl; // 구현용 구조체
    Impl *pImpl; // 구현용 구조체를 가리키는 포인터
};
```
- `Widget`이 `std::string`, `std::vector`, `Gadget`을 형식을 언급하지 않는다.
  - `Widget`의 클라이언트는 위 형식들의 헤더들을 `#include`로 포함시킬 필요가 없다.
- 컴파일 속도가 빨라지게 되었다.
- `Widget::Impl`처럼 선언만 하고 정의를 하지 않는 형식을 **불완전한 형식**이라고 한다.
  - `Pimpl` 관용구는 불완전 형식을 가리키는 포인터를 선언하는 것을 활용한다.

- 클래스를 구현하는 코드는 아래와 같다.
```cpp
#include"widget.h" // 구현 파일 widget.cpp 안에서
#include"gadget.h"
#include<string>
#include<vector>
struct Widget::Impl {           // 전에 Widget에 있던
    std::string name;           // 자료 멤버들을 담은
    std:;vector<double> data;   // Widget::Impl의 정의
    Gadget g1, g2, g3;
};

Widget::Widget()   // 이 Widget 객체를 위한
: pImpl(new Impl)  // 자료 멤버들을 할당
{}

Widget::~Widget() // 이 객체를 위한 자료
{ delete pImpl; } // 멤버들을 파괴
```
- `new`와 `delete`를 **직접 호출**하는 것이 너무 원시적이다.
  - `delete`를 사용한 해제가 없을 경우 누수가 발생할 가능성이 존재한다.
  - `Widget` 생성자에서 `Widget::Impl` 객체를 할당하고 `Widget`이 파괴될 때 객체를 해제해야 한다.
  - `std::unique_ptr`를 사용하기 좋은 상황이다.

### unique_ptr를 사용하여 pImpl 관용구 구현
```cpp
// Widget.h 파일
class Widget{
public:
    Widget();
private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
}

// Widget.cpp 파일
struct Widget::Impl {
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};

Widget::Widget()
: pImpl(std::make_unique<Impl>())
{}
```
- 생성시 [항목21](/Chapter4/Item21.md)에 조언을 따라 `make` 함수를 사용하여 `std::unique_ptr`를 만든다.
- 소멸자에 집어넣을 코드가 없기 때문에 소멸자가 없다.
  - 스마트 포인터의 이점 중 하나로 `unique_ptr`가 자신이 가리키는 객체를 삭제하기 때문에 이 클래스에서 따로 삭제할 것은 없다.
- `Widget`을 선언하는 코드에서 **컴파일 오류**가 발생한다.
```cpp
#include "widget.h"
Widget w;
```
- `w`가 파괴되는 지점에 `w`의 소멸자가 호출되는데 `std::unique_ptr`를 이용하는 `Widget`클래스에 따로 소멸자를 선언하지 않았다.
  - [컴파일러가 생성하는 특수 멤버 함수 규칙](/Chapter3/Item17.md)에 의해 컴파일러가 대신 소멸자를 작성한다.
  - 컴파일러는 `pImpl`의 소멸자를 호출하는 코드를 삽입한다.
  - 기본 삭제자를 사용하는 `std::unique_ptr`이고 기본 삭제자는 `std::unique_ptr` 안에 있는 `raw` 포인터에 대해 `delete`를 적용한다.
- 삭제자 함수는 `delete`를 적용하기 전에 `raw` 포인터가 불완전한 형식을 가리키지는 않는지 점검한다.
  - `Impl`이 불완전한 형식이기 때문에 오류가 발생하게 된다.
- 해결법은 **구현부**에서 **소멸자**를 선언하면 된다.
```cpp
// Widget.h 파일
class Widget{
public:
    Widget();
    ~Widget(); // 선언만 해둔다.

private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
}

// Widget.cpp 파일
#include"Widget.h"
#include"gadget.h"
#include<string>
#include<vector>

struct Widget::Impl {
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};

Widget::Widget()
: pImpl(std::make_unique<Impl>())
{}

Widget::~Widget() = default; // 소멸자 정의
```
- 소멸자의 정의가 구현 파일 안에서 생성되게 만드는 것이 소멸자를 선언한 유일한 이유임을 강조하기 위해 `=default`로 정의하였다.

## 이동과 복사 작업
- 복사 연산(깊은 복사)을 지원하는 것도 마찬 가지이다.
- [Widget에 소멸자를 선언하면 컴파일러는 이동 연산들을 작성하지 않는다.](/Chapter3/Item17.md)
  - 이동을 지원하려면 **이동 연산들을 직접 선언**해야 한다.
  - 이동 연산들도 소멸자처럼 정의를 **구현파일에 작성**해야 한다.

```cpp
// Widget.h 파일
class Widget{
public:
    Widget();
    ~Widget();
    Widget(Widget&& rhs);               // 선언만 하고
    Widget& operator=(Widget&& rhs);    // 정의는 하지 않는다.

    Widget(const Widget& rhs);
    Widget& operator=(const Widget& rhs);
private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
}
```

```cpp
// Widget.cpp 파일
#include"Widget.h"
#include"gadget.h"
#include<string>
#include<vector>

struct Widget::Impl {
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};

Widget::Widget()
: pImpl(std::make_unique<Impl>())
{}

Widget::~Widget() = default;

Widget::Widget(Widget&& rhs) = default;             // 여기서
Widget& Widget::operator=(Widget&& rhs) = default;  // 정의

Widget::Widget(const Widget& rhs) : pImpl(nullptr)
{
    if (rhs.pImpl) pImpl = std::make_unique<Impl>(*rhs.pImpl); }
}

Widget& Widget::operator=(const Widget& rhs)
{
    if (!rhs.pImpl) pImpl.reset();
    else if (!pImpl) pImpl = std::make_unique<Impl>(*rhs.pImpl);
    else *pImpl = *rhs.pImpl;
    return *this;
}
```

## shared_ptr를 pImpl로 사용한다면 ?
```cpp
// widget.h
class Widget
{
public:
    Widget();
    // 소멸자나 이동 연산들의 선언이 전혀 없음
private:
    struct Impl;
    std::shared_Ptr<Impl> pImpl; // std::shared_ptr 사용
};

Widget w1;
auto w2(std::move(w1)); // w2를 이동 생성
w1 = std::move(w2);     // s1을 이동 할당
```
- `std::unique_ptr`와는 다르게 구현부를 `Impl` 구조체보다 아래 두는 등의 처리를 하지 않아도 컴파일되고 잘 동작한다. 
- 차이가 발생하는 이유
  - `std::unique_ptr`의 타입에는 커스텀 삭제자의 타입이 포함되지만 `std::shared_ptr`의 타입에는 포함되지 않는다.
    - `std::unique_ptr`는 삭제자의 타입이 포함되기 때문에 컴파일러가 자동으로 함수들을 생성할 때 불완전한 타입이 포함되어 있으면 안 된다. 
    - `std::shared_ptr`는 삭제자의 타입이 포함되지 않기 때문에 컴파일러가 자동으로 함수를 생성할 때 삭제자의 타입이 불완전해도 괜찮다.
