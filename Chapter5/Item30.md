# 항목 30. 완벽 전달이 실패하는 경우들을 잘 알아두라
## 완벽 전달
- `C++`의 완벽 전달은 이름만큼 완벽하지는 않다.
- 한 함수가 자신의 인수들을 다른 함수에 완벽히 전달하는 것을 의미한다.
- 완벽 전달은 단순히 객체를 전달하는 것만이 뿐만 아니라 왼값 또는 오른값 여부, `const`, `volatile` 여부 까지도 전달한다.
- 이를 위해서는 **보편 참조 매개변수**가 필요하다.
  - 전달된 인수의 왼값, 오른값에 대한 정보를 부호화하는 것은 보편 참조 매개변수가 유일하기 때문이다.
- 완벽 전달을 수행하는 함수는 다음과 같은 형태를 가진다.
```cpp
template<typename T>
void fwd(T&& param)               // 임의의 인수를 받는다.
{
  f(std::forward<T>(param));      // 인수를 f에 전달한다.
}

template<typename... Ts>
void fwd(Ts&&... params)          // 임의의 인수들을 받는다.
{
  f(std::forward<T>(params)...);  // 그것들을 f에 전달한다.
}
```
- [표준 컨테이너의 emplace류 함수들](/Chapter8/Item42.md)과 [스마트 포인터 팩토리 함수 std::make_shared, std::make_unique](/Chapter4/Item21.md)에서 이런 형태를 볼 수 있다.
- 대상 함수 `f`와 전달 함수 `fwd`가 있을떄 같은 인수로 `f`를 호출했을 때 일어나는 일과 `fwd`를 호출했을 때 일어나는 일이 다르다면 **완벽 전달은 실패**한 것이다.

## 완벽 전달이 실패하는 인수의 종류
### 중괄호 초기치
```cpp
void f(const std::vector<int>& v);

f({ 1, 2, 3});      // 컴파일 성공 {1, 2, 3}은 암묵적으로 std::vector<int>로 변환된다.

fwd({ 1, 2, 3 });   // 컴파일 실패
```
- `f`를 직접 호출하면 `std::initializer_list` 으로부터 **암묵적 변환**을 수행해 `std::vector<int>` 객체를 생성하여 컴파일에 성공한다.
- `fwd`를 통해 `f`를 간접적으로 호출할 때에는 상황이 다르다.
- 컴파일러가 `fwd`의 호출 지점에서 전달된 인수들과 `f`에 선언된 매개변수를 직접 비교할 수 없다.
- 컴파일러는 `fwd`에 전달되는 인수들의 형식을 연역하고, 연역된 형식들을 `f`의 매개변수 선언들과 비교한다.
- 비교할 때 아래 조건 중 하나라도 만족하면 완벽 전달이 실패한다.
  - `fwd`의 매개변수들 중 하나 이상에 대해 **컴파일러가 형식을 연역하지 못하는 경우**
  - `fwd`의 매개변수들 중 하나 이상에 대해 **컴파일러가 형식을 잘못 연역하는 경우**
- `fwd({1, 2, 3})`호출의 문제는 `std::initializer_list`가 될 수 없는 형태로 선언된 함수 템플릿 매개변수에 **중괄호 초기치**를 넘겨준다는 것이다.
  - 표준에서는 **비연역 문맥**이라고 부른다. 

#### 문제 해결 방법
- [auto 변수는 중괄호 초기치로 초기화해도 연식이 잘 연역된다.](/Chapter1/Item2.md)
  - 그런 변수는 `std::initializer_list`객체로 간주된다.
- `auto`를 이용해서 지역 변수를 선언한 후 지역 변수를 전달함수에 넘겨주는 우회책을 사용할 수 있다.
```cpp
auto il = {1, 2, 3}; // il의 형식 연역 결과는 std::initializer_list<int>이다.
fwd(il);             // il이 f로 완벽하게 전달된다.
```

### 널 포인터를 뜻하는 0 또는 NULL
- 0이나 `NULL`을 널 포인터로서 템플릿에 넘겨주려 하면 컴파일러가 그것은 **포인터 형식이 아닌 정수 형식으로 인식**하기 때문에 문제가 생긴다.
  - 0과 `NULL`은 널 포인터로서 완벽하게 전달되지 못한다.

#### 문제 해결 방법
- 0이나 `NULL`대신 [nullptr](/Chapter3/Item8.md)를 사용하면 된다.

### 선언만 된 정수 static const 및 constexpr 자료 멤버
- 정수 `static const` 멤버와 정수 `static constexpr` 멤버는 클래스 안에서 정의할 필요 없이 **선언**만 하면 된다.
  - 컴파일러가 `const` 전파를 적용해서 값을 위한 메모리를 따로 마련할 필요가 없어지기 때문이다.
```cpp
class Widget {
public:
  static constexpr std::size_t MinVals = 28; // MinVals의 선언
  ...
};
...                                          // MinVals의 정의는 없음

std::vector<int> widgetData;
widgetData.reserve(Widget::MinVals);         // Minvals 사용
```
- 위 코드는 `Widget::MinVals`를 이용해서 `widgetData`의 초기 용량을 설정한다.
- `MinVals`의 **정의가 없어도 이 코드는 잘 동작**한다.
  - 컴파일러가 `MinVals`가 언급된 모든 곳에 `28`이라는 값을 배치하기 때문이다.
- 어떤 코드가 `MinVals`의 주소를 취한다면 `MinVals`를 위한 장소가 필요해진다.
  - 코드는 컴파일되긴 하지만 `MinVals`의 정의가 없어서 **링크에 실패**한다.

```cpp
void f(std::size_t val);

f(Widget::MinVals);     // OK

fwd(Widget::MinVals);   // 링크 실패
```
- `f`를 `MinVals`로 호출하는 것은 문제가 없다.
  - 컴파일러가 `MinVals`를 해당 값으로 대체해준다.
- `fwd`를 거쳐서 `f`를 호출하려 하면 컴파일은 되지만 **링크에 실패**한다.
- `fwd`의 매개변수는 **보편 참조**이며 컴파일러가 산출한 코드에서 참조는 **포인터**처럼 취급된다.
- 바이너리코드에서 포인터와 참조는 본질적으로 같다.
  - 참조는 **포인터처럼 취급**되다가 사용할 때만 자동으로 **역참조 연산**을 수행해준다.
- `MinVals`를 참조로 전달하는 것은 사실상 포인터로 넘겨주는 것이다.
  - `MinVals`의 주소를 참조하다가 링킹 에러를 발생시키게 된다.

#### 문제 해결 방법
- `CPP` 파일에 **정의부를 구현**해주면 된다.
```cpp
constexpr std::size_t Widget::MinVals; // Widget의 .cpp 파일에서
```

### 중복적재된 함수 이름과 템플릿 이름
- `f`가 하나의 **함수를 인자**로 받아서 그 함수를 호출한다고 가정한다.
  - 그 함수는 `int`를 받고 `int`를 돌려준다.
```cpp
void f(int (*pf)(int)); // pf는 processing function을 뜻한다.
```
- `f`를 더 간단한 **비 포인터 구문**으로 선언할 수 있다.
```cpp
void f(int pf(int));
```
- 아래와 같이 오버로딩된 `processVal`함수가 있을 때 `processVal`을 `f`에 넘겨주는 것이 가능하다.
```cpp
int processVal(int value);
int processVal(int value, int priority);

f(processVal);   // OK
fwd(processVal); // 오류! 어떤 processVal인지?
```
- `f`에 전달하는 `processVal`은 첫번째 `processVal`가 적절하므로 **첫번째가 선택**된다.
- 함수 템플릿인 `fwd`에는 호출에 필요한 형식에 관한 정보가 전혀 없기 때문에 컴파일러는 어떤 오버로딩을 선택해야 할지 결정하지 못한다.
- `processVal`자체에는 형식이 없으며 형식이 없으면 형식 연역도 없다.
  - **형식 연역이 없다**는 점이 **완벽 전달이 실패**하는 또 다른 경우이다.
- 오버로딩된 함수 이름 대신 **함수 템플릿**을 사용하려 할 경우에도 같은 문제가 발생한다.
  - 함수 템플릿은 하나의 함수를 나타내는 것이 아니라 다수의 함수를 대표한다.
```cpp
template<typename T>
T workOnVal(T param) // 값들을 처리하는 템플릿 함수
{ ... }

fwd(workOnVal);      // 오류! workOnVal의 어떤 인스턴스 인지?
```
- `workOnVal`이라는 이름만 가지고는 어떻게 인스턴스화해야하는지 알아낼 방법이 없다.

#### 문제 해결 방법
- 타입이 없어서 연역을 하지 못해 발생한 문제이므로 **타입을 지정**해주면 문제를 해결할 수 있다.
```cpp
using ProcessFuncType = int (*)(int);           // typedef들을 만든다.
  
ProcessFuncType processValPtr = processVal;     // processVal에 필요한 서명을 명시한다.
  
fwd(processValPtr);                             // OK
fwd(static_cast<ProcessFuncType>(workOnVal));   // OK
```

## 비트필드
- 비트필드가 함수 인수로 쓰일때도 완벽 전달이 실패한다.
- 실제 응용에서 이것이 어떤 의미인지 보여주는 한 예로 `IPv4` 헤더를 나타내는 구조체이다.
```cpp
struct IPv4Header {
  std::uint32_t version:4,
                IHL:4,
                DSCP:6,
                ECN:2,
                totalLength:16;
  ...
};

void f(std::size_t sz); // 호출할 함수

IPv4Header h;

f(h.totalLength);       // OK
fwd(h.totalLength);     // error
```
- 문제는 `fwd`의 매개변수가 참조이고 `h.totalLength`는 비 `const` 비트필드라는 점이다.
- `C++` 표준은 둘의 조합에 대해 **비const 참조는 절대로 비트필드에 묶이지 않아야 한다**는 조건이 있다. 
- 비트필드들은 컴퓨터 워드의 일부분으로 존재할 수 있는데, 그런 일부 비트를 직접적으로 지칭하는 방법은 없다.
  - 임의의 비트들을 가리키는 포인터를 생성하는 방법은 없으며, 따라서 참조를 임의의 미트들에 맊눈 방법도 없다.
- 비트필드를 **값으로 전달**하거나 **const에 대한 참조**로 전달하는 건 가능하다.
  - 전달 대상 함수는 항상 비트필드의 **값의 복사본**을 받게 된다.

#### 문제 해결 방법
- 해당 **복사본을 직접 생성**해서 그 **복사본으로 전달함수를 호출**한다.
```cpp
// 비트필드 값을 복사한다
auto length = static_cast<std::uint16_t>(h.totalLength);
fwd(length);
```