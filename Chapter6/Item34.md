# 항목 34. std::bind보다 람다를 선호하라
## 가독성 문제
- 가장 중요한 이유는 람다가 가독성이 더 좋다는 것이다.
  - `C++11`에서는 람다가 거의 항상 `std::bind`보다 낫다.
  - `C++14`에서는 람다가 확고하게 우월하다.
- 알람 함수를 작성하는 코드가 있다.
```cpp
// 시간상의 한 지점을 대표하는 형식 별칭
using Time = std::chrono::steady_clock::time_point;

// enum class
enum class Sound { Beep, Siren, Whistle };

// 시간 길이를 나타내는 형식에 대한 별칭 선언
using Duration = std::chrono::steady_clock::duration;

// 시간 t에서 소리 s를 기간 d만큼 출력한다.
void setAlarm(Time t, Sound s, Duration d);
```
- 위 함수를 이용해서 한 시간 후부터 30초간 소리를 내는 알람 함수 작성해야 한다.

### 람다 사용
- 람다를 이용해서 만들면 다음과 같이 만들 수 있다.
```cpp
auto setSoundLambda =
  [](Sound s)
  {
    using namesapce std::chrono;
    setAlarm(steady_clock::now() + hours(1), s, seconds(30));
  };
```
- `C++14`에서는 `C++11`의 사용자 정의 리터럴 기능에 기초하는 **표준 접미사**들을 이용해 코드를 더욱 간결하게 작성할 수 있다.
  - 초는 `s`, 밀리초는 `ms`, 시간은 `h`라는 **접미사**를 이용해서 지정하면 된다.
```cpp
auto setSoundLambda =
  [](Sound s)
  {
    using namesapce std::chrono;
    using namespace std::literals;
    setAlarm(steady_clock::now() + 1h, s, 30s);
  };
```

### std::bind 사용
- 이 함수를 바인드를 이용해서 만들면 다음과 같이 만들어진다.
  - 아래 함수는 오류가 포함되어 있다.
```cpp
using namespace std::chrono;
using namespace std::literals;

using namespace std::placeholders; // _1을 사용하는 데 필요함  

auto setSoundBind = std::bind(setAlaram, steady_clock::now() + 1h, _1, 30s);
```
- `setSoundBind`를 호출하면 `std::bind` 호출에 지정된 시간과 기간으로 `setAlarm`이 호출된다.
- _1은 자리표로 `setSoundBind`의 `operator()`를 호출할 때 들어가는 인자에 순서대로 이름을 붙인 것이라고 생각하면 된다.
- `setSoundBind` 호출의 **첫 인자**가 `setAlarm`의 **둘째 인자**로 전달된다는 점을 파악하려면 자리표의 번호를 `std::bind` 매개변수 목록에서의 자리표의 위치와 연결하는 과정을 거쳐야 한다.
- `std::bind` 호출만 봐서는 인자의 형식을 알 수 없으므로 `setSoundBind`에 어떤 종료의 인수가 전달되는지 알려면 `setAlarm` 선언을 봐야 한다.
- 람다 버전에서는 `steady_clock::now() + 1h`라는 표현식이 `setAlarm`에 전달되는 **하나의 인자임이 명백**하다.
  - 표현식을 `setAlarm`이 호출되는 시점에서 평가된다.
  - `setAlarm`을 호출한 시점에서부터 한 시간이 지난 후에 경보가 울려야 한다는 점에서 합당한 일이다.
- `std::bind` 호출에서 `steady_clock::now() + 1h`라는 표현식은 `setAlarm`이 아니라 `std::bind`로 전달된다.
  - 표현식이 평가되어서 나오는 시간이 `std::bind`가 생성한 바인드 객체에 저장된다.
  - 경보는 `setAlarm`을 호출하고 한 시간이 지난 후가 아니라 **std::bind를 호출하고 한 시간이 지난 후**에 울리게 된다.

### 문제의 해결
- 표현식을 `setAlarm` 호출 때까지 지연하라고 `std::bind`에 알려주어야 한다.
  - 첫 `std::bind` 안에 두 개의 함수 호출을 내포시켜야 한다.
```cpp
// C++11
using namespace std::chrono;
using namespace std::placeholders;

auto setsoundBind =
  std::bind(setAlarm, std::bind(std::plus<>(steady_clock::time_point>(),
                      std::bind(steady_clock::now), hours(1)), _1, seconds(30)));

// C++14
auto setsoundBind = std::bind(setAlarm, std::bind(std::plus<>(), std::bind(steady_clock::now), 1h), _1, 30s);
```
- `C++11` 코드만 봐도 가독성 측면에서 람다를 선호해야할 이유가 명백하다.

## 오버로딩 문제
- `setAlarm` 함수에서 경보음의 크기를 네 번째 매개변수로 받는 **오버로딩 버전**을 추가해본다.
```cpp
enum class Volume {Normal, Loud, LoudPlusPlus};
void setAlarm(Time t, Sound s, Duration d, Volume v);
```
- 람다를 사용하면 문제 없이 작동한다.
```cpp
auto setSoundLambda = 
  [] (Sound s)
  {
    using namespace std::chrono;
    setAlarm(steady_clock::now(), 1h, s, 30s)
  };
```
- std::bind 버전은 컴파일되지 않는다.
```cpp
auto setSoundBind =
  std::bind(setAlarm,
            std::bind(std::plus<>(),
                      std::bind(steady_clock::now), 1h), _1, 30s);
```
- 컴파일러는 두 `setAlarm` 함수 중 어떤 것을 `std::bind`에 넘겨주어야 할지 **결정할 방법이 없다.**
- 컴파일에 성공하려면 `setAlarm`을 적절한 **함수 포인터 형식으로 캐스팅**해야 한다.
```cpp
using SetAlram3ParamType = void(*)(Time t, Sound s, Duration d);

auto setSoundBind =
  std::bind(static_cast<SetAlram3ParamType>(setAlarm),
            std::bind(std::plus<>(),
                      std::bind(steady_clock::now), 1h), _1, 30s);
```
- 위처럼 캐스팅을 사용하면 컴파일에 성공은 하나 람다와 `std::bind`의 차이점이 하나 더 생기게 된다.
- 람다의 경우에는 `setAlarm` 호출을 인라인화 시키는데 반해 바인드의 경우에는 **함수 포인터**를 사용하기 때문에 인라인화할 가능성이 매우 낮다.
  - `std::bind`를 사용할 때 보다 람다를 사용할 때 더 빠른 코드가 산출될 수 있다.

## 복잡한 상황에서의 람다의 이점
- 코드가 복잡해질수록 바인드로의 구현이 어렵다.
- 어떤 인수가 범위 안에 있는지 확인하는 예제가 있다.
```cpp
//C++11 lambda
auto betweenLambda =
  [lowVal, highVal]
  (int val)
  { return lowVal <= val && val <= highVal; };

//C++14 lambda
auto betweenLambda =
  [lowVal, highVal]
  (const auto& val)
  { return lowVal <= val && val <= highVal; };
  
//C++11 bind
auto betweenB = 
    std::bind(std::logical_and<bool>(),
              std::bind(std::less_equal<int>(),lowVal, _1),
              std::bind(std::less_equal<int>(), _1, highVal);

//C++14 bind
auto betweenB = 
    std::bind(std::logical_and<>(),
              std::bind(std::less_equal<>(),lowVal, _1),
              std::bind(std::less_equal<>(), _1, highVal);
```
- 둘 사이만 봐도 람다가 더 간편하고 이해하기 쉽다는 것을 알 수 있다.
  - `std::bind`로 짠 코드는 일부로 남들이 이해하지 못하게 작성한 코드처럼 보인다.
- 람다 버전이 더 짧고 이해하기 쉬우며 유지보수하기도 쉽다.

## 참조와 복사의 문제
- `std::bind`의 문제로 자리표의 행동이 불투명한 것만 있는것은 아니다.
- `Widget` 객체를 압축 복사본을 만드는 함수가 있다.
```cpp
enum class CompLevel { Low, Normal, High };       // 압축 수준
Widget compress(const Widget& w, CompLevel lev);  // w의 압축 복사본을 만든다.
```
- 특정한 `Widget` 객체 `w`의 압축 수준을 지정할 수 있는 함수 객체를 만들어 본다.
- `std::bind`를 사용하면 아래와 같다.
```cpp
Widget w;
using namespace std::placeholders;
auto compressRateBind = std::bind(compress, w, _1);
```
- `std::bind`에 전달된 `w`는 이후의 `compress` 호출을 위해 반드시 바인드 객체 `compressRateBind`안에 저장된다.
- `w`는 값으로 저장되는데 바인드의 작동 방식을 모른다면 추론은 불가능하다.
- 람다의 경우 `w`가 **값**으로 갈무리되는지 **참조**로 갈무리되는지가 소스 코드에 명백히 드러난다.
```cpp
auto compressRateLambda =        // w는 값으로 갈무리되고
  [w](CompLevel lev)             // lev는 값으로 전달된다.
  { return compress(w, lev); };
```
- 람다에서는 매개변수들이 전달되는 방식도 갈무리만큼이나 명백하게 드러난다.
- `std::bind`로 얻은 바인드 객체를 호출할 때에는 인수의 전달 방식이 명확하지 않다.

## std::bind를 사용해야하는 상황
- `C++14`에서는 `std::bind`를 사용하는 것이 합당한 경우가 없지만 `C++11`에서는 아래 제한된 두 경우에서는 `std::bind`의 사용을 정당화 할 수 있다.
### 이동 갈무리
- [C++11 람다는 이돌 갈무리를 지원하지 않는다.](/Chapter6/Item32.md)
- `C++14`에서는 초기화 갈무리 덕분에 흉내가 필요하지 않다.

### 다형적 함수 객체
- 바인드 객체에 대한 함수 호출 연산자는 완벽 전달을 사용하기 때문에 그 어떤 형식의 인수들도 받을 수 있다.
  - 객체를 템플릿화된 함수 호출 연산자와 묶으려 할 때 유용하다.

```cpp
class PolyWidget {
public:
  template<typename T>
  void operator()(const T& param) const;
  ...
};

```
- `std::bind`로 `PolyWidget` 객체를 아래처럼 묶을 수 있다.
```cpp
PolyWidget pw;
auto boundPW = std::bind(pw, _1);
```
- `boundPW`를 서로 다른 형식의 인수들로 호출할 수 있다.
```cpp
boundPW(1930);      // PolyWidget::operator()에 int를 전달
boundPW(nullptr);   // PolyWidget::operator()에 nullptr를 전달
boundPW("Rosebud"); // PolyWidget::operator()에 문자열을 전달
```
- `C++11`의 람다로는 이런 일이 불가능하고 `C++14`에서는 `auto` 매개변수를 가진 람다로 구현할 수 있다.
```cpp
auto boundPW = [pw](const auto& param) { pw(param); };
```