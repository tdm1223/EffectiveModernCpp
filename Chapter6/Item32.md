# 항목 32. 객체를 클로저 안으로 이동하려면 초기화 갈무리를 사용하라
## 객체를 클로저 안으로 들여올 경우
- 값 갈무리, 참조 갈무리 모두 마땅치 않다.
- `C++11`에서는 방법이 없지만 `C++14`에서 **객체를 클로저 안으로 이동**하는 수단을 제공한다.
- `C++11`에서는 흉내내는 방법이 존재한다.

## 초기화 갈무리
- `C++11`의 갈무리 모드들이 할 수 있는 모든 것을 할 수 있으며 그 외의 여러 가지 것들도 할 수 있다.
- 기본 갈무리 모드는 표현할 수 없지만 어차피 [기본 갈무리 모드는 피해야 한다.](/Chapter6/Item31.md)

### 초기화 갈무리로 지정할 수 있는 것
1. 람다로부터 생성되는 클로저 클래스에 속한 **자료 멤버의 이름**
2. 그 자료 멤버를 초기화하는 **표현식**

### 초기화 갈무리 예
- 초기화 갈무리를 이용해서 `std::unique_ptr`를 **클로저 안으로 이동**하는 예이다.
```cpp
class Widget {
public:
  ...
  bool isValidated() const;
  bool isProcessed() const;
  bool isArchived() const;
  
private:
  ...
};

auto pw = std::make_unique<Widget>();         // Widget을 생성한다

...                                           // 여기서 pw를 조작한다.

auto func = [pw = std::move(pw)]              // 클로저의 자료 멤버를
            { return pw->isValidated()        // std::move(pw)로
                     && pw->isArchived(); };  // 초기화한다.
```
- `pw = std::move(pw)` 부분이 초기화 갈무리이다.
  - 좌변(`pw`)은 클로저 클래스 안의 자료 멤버의 이름이다.
  - 우변(`std::move(pw)`)은 좌변을 초기화하는 표현식이다.
- 좌변과 우변의 범위가 다르다.
  - 좌변의 범위는 해당 **클로저 클래스의 범위**이고 우변의 범위는 **람다가 정의되는 지점의 범위**와 동일하다.
  - 좌변의 이름은 클로저 클로저 클래스 안의 자료 멤버를 지칭하지만 우변의 이름은 람다 이전에 선언된 객체를 의미한다.
- 클로저 안에서 자료 멤버 `pw`를 생성하되, 지역 변수 `pw`에 `std::move`를 적용한 결과로 자료 멤버를 초기화하라는 뜻이다.
- `std::make_unique`로 생성한 직후의 `Widget`이 이미 람다로 갈무리하기에 적합한 상태라면 직접 초기화 할 수 있다.
```cpp
auto func = [pw = std::make_unique<Widget>()]  // 클로저 안의 자료 멤버를
            { return pw->isValidated()         // make_unique 호출 결과로
                     && pw->isArchived(); };   // 초기화한다.
```
- 위 코드는 `C++11`에서는 불가능 하고 `C++14`에서 가능하다.
- **초기화 갈무리**를 **일반화된 람다 갈무리**라고 부르기도 한다.

## C++11에서 초기화 갈무리
### 클래스를 직접 생성
- 람다 표현식은 컴파일러가 하나의 클래스를 자동으로 작성해서 클래스의 객체를 생성하게 만드는 수단일 뿐이다.
- ``C++11``에서는 **클래스를 직접 생성**하는 식으로 작성이 가능하다.
```cpp
class IsValAndArch {
public:
  using DataType = std::unique_ptr<Widget>;
  
  explicit IsValAndArch(DataType&& ptr)
  : pw(std::move(ptr)) {}
  
  bool operator()() const
  { return pw->isValidated() && pw->isArchived();}
  
private:
  DataType pw;
};

auto func = IsValAndArch(std::make_unique<Widget>());
```
- 작성해야할 코드가 많지만 `C++11`에서도 자료 멤버의 이동 초기화를 지원하는 클래스를 작성할수 있다.

### 이동 갈무리 흉내내기
- 람다 표현식과 `std::bind`를 적절히 응용하면 `C++11`에서도 `C++14` 스타일의 갈무리를 흉내낼 수 있다.
  - 갈무리할 객체를 `std::bind`가 산출하는 함수 객체로 이동한다.
  - **갈무리된 객체에 대한 참조**를 람다에 넘겨준다.
- 지역 `std::veoctor`를 생성해서 값을 추가한 후 클로저 안으로 이동하는 코드이다.
```cpp
std::vector<double> data;                 // 클로저 안으로 이동할 객체

auto func = [data = std::move(data)]      // C++14의 초기화 갈무리   
            { /* 여기서 data를 사용 */};

auto func =
  std::bind(                              // C++11에서 초기화 갈무리를
    [](const std::vector<double>& data)   // 흉내 내는 방법
    { /* 여기서 data를 사용 */ },
    std::move(data)
  );
```
- 람다 표현식처럼 `std::bind`는 **함수 객체를 산출**한다.
- `std::bind`가 리턴하는 함수 객체를 **바인드 객체**라한다.
- `std::bind`의 첫번째 인자는 호출 가능한 객체이고 나머지 인자들은 그 객체에 전달할 값들을 나타낸다.
- 바인드 객체는 `std::bind`에 전달된 모든 인자의 복사본들을 포함한다.
  - 왼값 인수에 대해 바인드 객체에는 **복사 생성된 객체**가 있다.
  - 오른값 인수에 대해 바인드 객체에는 **이동 생성된 객체**가 있다.
- 두번째 인자는 `std::move`의 결과이므로 오른값이고 `data`는 바인드 객체 안으로 이동한다.
  - 이동 생성이 이동 객체 흉내의 핵심이다.
  - 오른값을 바인드 객체 안으로 이동함으로써 오른값의 이동이 불가능하다는 `C++11` **클로저의 한계를 우회**한다.
- 바인드 객체가 호출되면 바인드 객체에 저장된 인자들이 `std::bind` 호출 시 첫 인자로 지정한 호출 가능한 객체에 전달된다.
  - `func`가 호출되면 `func`에 저장된 `data`의 복사본(이동 생성된)이 `std::bind` 호출 시 지정한 람다의 인자로서 전달된다.
- 이 람다는 `C++14` 버전에서 사용한 람다와 같지만 이동 갈무리 흉내용 객체에 해당하는 **data 매개변수가 추가**되었다는 차이가 있다.
  - 이 매개변수는 바인드 객체 안의 `data` 복사본에 대한 **왼값 참조**이다.
  - 람다 본문 안에서 `data`를 사용하는 코드는 바인드 객체 안의 이동 생성된 `data`의 **복사본**을 사용하게 된다.
- 기본적으로 람다로부터 만들어진 클로저 클래스 내부에 생성되는 `operator()` 함수는 `const`이다.
  - 람다 본문 안에서 클로저의 모든 멤버는 `const`가 된다.
- 바인드 객체 안의 이동 생성된 `data` 복사본은 `const`가 아니다.
  - 람다 안에서 `data` 복사본이 수정되지 않게 하기 위해 람다의 매개변수를 **const에 대한 참조**로 선언하여야 한다.
- 람다를 `mutable`로 선언하면 클로저 클래스의 `operator()`는 `const`로 선언되지 않을 것이므로 람다의 매개변수 선언에서 `const`를 제거해야한다.
```cpp
auto func =
    std::bind(                                  // C++11에서 mutable
        [](std::vector<double>& data) mutable   // 람다의 초기화 갈무리를
        { }, std::move(data));                  // 흉내 내는 방법
```
- 바인드 객체는 `std::bind`에 전달된 모든 인자의 복사본을 저장하므로 이 예의 바인드 객체는 람다가 산출한 클로저의 복사본도 저장한다.
- 클로저의 수명은 바인드 객체의 수명과 같다.
- 클로저가 존재하는 한 이동 갈무리를 흉내 내는 객체를 담은 바인드 객체도 존재함을 뜻한다.

## 요약
- 객체를 `C++11` 클로저 안으로 이동 생성하는 것은 불가능하나, 객체를 `C++11` 바인드 객체 안으로 이동 생성하는 것은 가능하다.
- `C++11`에서 이동 갈무리를 흉내내는 방법은 객체를 바인드 객체 안으로 **이동 생성**하고, 이동 생성된 객체를 람다에 **참조로 전달**하는 것이다.
- 바인드 객체의 수명이 클로저의 수명과 같으므로 바인드 객체 안의 객체들을 망치 클로저 안에 있는 것처럼 취급하는 것이 가능하다.
- [std::bind보다는 람단를 선호하는것이 좋지만](/Chapter6/Item34.md) `std::bind`가 유용한 경우도 존재한다.