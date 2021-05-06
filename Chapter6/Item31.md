# 항목 31. 기본 갈무리 모드를 피하라
- 람다를 이용하면 함수 객체를 쉽게 만들 수 있다.
- 람다는 **콜백 함수**나 **인터페이스 적응 함수**, 단발성 호출을 위한 **문맥 국한적 함수**를 지정할때 유용하다.
- 람다 덕분에 `C++`이 좀 더 쾌적한 프로그래밍 언어가 된다.

## 람다 기본 용어
### 람다 표현식
- 하나의 표현식으로 **소스코드의 일부**이다.
```cpp
std::find_if(container.begin(), container.end(),
             [](int val) { return 0 < val && val < 10; }); // 내부에 존재하는 식이 람다 표현식
```

### 클로저
- 람다에 의해 만들어진 **실행시점 객체**이다.
- 갈무리 모드에 따라 클로저가 갈무리된 자료의 복사본을 가질 수도 있고 그 자료에 대한 참조를 가질 수도 있다.
- `std::find_if` 호출에서 클로저는 실행시점에서 `std::find_if`의 **셋째 인수로 전달되는 객체**이다.

### 클로저 클래스
- 클로저를 만드는 데 쓰인 클래스를 말한다.
- 각각의 람다에 대해 컴파일러는 고유한 클로저 클래스를 만든다.
- 람다 안의 문장들은 해당 클로저 클래스의 멤버 함수들 안의 실행 가능한 명령들이 된다.

```cpp
{
    int x;
    auto c1 = [x](int y) { return x * y > 55; };

    auto c2 = c1;
    auto c3 = c2;
}
```
- c1, c2, c3은 모두 람다가 산출한 클로저의 복사본이다.

## 기본 갈무리 모드
- `C++11`에서 기본 갈무리 모드는 두 가지로 참**조에 의한 갈무리 모드**와 **값에 의한 갈무리 모드**이다.
- 기본 참조 갈무리에서는 참조가 대상을 잃을 위험이 있다.
- 기본 값 갈무리에서는 두 가지 오해를 유발한다는 점에서 위험하다.
  - 참조가 대상을 잃는 문제가 없을 것 같지만 그렇지 않다.
  - **자기완결적**일 것 같지만 그렇지 않은 경우도 있다.

## 참조에 의한 갈무리 모드의 위험성
- 참조 갈무리를 사용하는 클로저는 **지역 변수** 또는 람다가 정의된 범위에서 볼 수 있는 **매개변수**에 대한 참조를 가지게 된다.
- 람다에 의해 생성된 클로저의 수명이 지역 변수나 매개변수의 수명보다 오래 지속되면 클로저 안의 참조는 대상을 잃는다.
```cpp
using FilterContainer = std::vector<std::function<bool(int)>>;
  
FilterContainer filters; // 필터링 함수들

void addDivisorFilter()
{
  auto calc1 = computeSomeValue1();
  auto calc2 = computeSomeValue2();
  auto divisor = computeDivisor(calc1, calc2);
  filters.emplace_back(
    [&](int value) { return value % divisor == 0; } // 위험
  );
}
```
- `int` 하나를 받아서 그 값이 필터를 만족하는 지를 뜻하는 `bool` 하나를 돌려주는 필터링 함수들을 담는 코드이다.
- 람다는 지역변수 `divisor`를 참조하는데 변수는 `addDivisorFilter`가 반환되면 더 이상 존재하지 않게 된다.
- `addDivisorFilter`는 `filters.emplace_back`이 반환된 직후에 반환되므로 `filters`에 추가되는 필터 함수는 이미 사망한 상태로 도착한다.
- 필터는 생성 직후부터 미정의 행동을 유발한다.
- `divisor`의 참조 갈무리를 **명시적으로 지정**해도 같은 문제가 발생한다.
```cpp
filters.emplace_back(
  [&divisor](int value)             // 위험!! 이번에도
  { return value % divisor == 0; }  // divisor 참조는 대상을 잃는다.
);
```
- 람다 표현식의 유효셩이 `divisor`의 수명에 의존한다는 점이 명확히 나타난다는 장점이 있다.

### 값 갈무리 모드를 통한 해결
- 한 가지 해결책은 값 갈무리 모드를 사용하는 것이다.
- `filters`에 람다를 아래와 같이 추가하면 된다.
```cpp
filters.emplace_back(
  [=](int value) { return value % divisor == 0; }
);
```
- 기본 값 갈무리 모드를 사용하면 위 예제는 해결할 수 있다.
- 예상만큼 안전하지는 않다.
- 포인터를 값으로 갈무리하면 그 포인터는 람다에 의해 생성된 클로저 안으로 복사된다.
  - 람다 바깥의 어떤 코드가 포인터를 `delete`로 삭제하지 않는다는 보장은 없다.
  - 그런일이 발생하면 포인터 복사본은 지칭 대상을 잃게 된다.

## 값에 의한 갈무리 모드의 위험성
### raw 포인터에 대한 문제
- Widget 클래스가 필터들의 컨테이너에 필터 함수를 추가하는 코드이다.
```cpp
class Widget {
public:
  ...
  void addFilter() const;
  
private:
  int divisor;
};

void Widget::addFilter() const
{
  filters.emplace_back(
    [=](int value) { return value % divisor == 0; }
  };
}
```
- 잘 모르는 사람에게는 안전한 코드로 보일 것이다.
- 람다는 `divisor`에 의존하는데 값 갈무리 모드에서는 `divisor`의 값이 람다가 생성하는 클로저 안으로 복사되므로 안전하지 않겠냐는 생각을 할것이다.
- 갈무리는 오직 람다가 생성된 범위 안에서 보이는 `static`이 아닌 **지역변수**(매개변수 포함)에만 적용된다.
- `Widget::addFilter`에서 `divisor`는 지역 변수가 아니라 클래스의 한 **멤버 변수**이므로 갈무리 될 수 없다.
- 기본 갈무리 모드를 뜻하는 `=`를 제거하면 코드는 컴파일 되지 않는다.
- `divisor`를 명시적으로 갈무리하려는 갈무리 절은 컴파일 되지 않는다.
  - `divisor`가 **지역 변수**도 아니고 **매개변수**도 아닌 **멤버 변수**이기 때문이다.
```cpp
// 기본 갈무리 모드를 뜻하는 =를 제거
void Widget::addFilter() const
{
  filters.emplace_back(
    [](int value) { return value % divisor == 0; } // 컴파일 오류
  );
}

// divisor를 명시적으로 갈무리 하려는 갈무리절
void Widget::addFilter() const
{
  filters.emplace_back(
    [divisor](int value)              // 오류! 갈무리할 지역
    { return value % divisor == 0; }  // divisor가 없음
  );
}
```

### 컴파일 오류가 발생하는 이유
- 암묵적으로 `raw` 포인터인 `this` 포인터가 쓰인다.
- 모든 비`static` 멤버 함수에는 `this` 포인터가 있으며 클래스의 멤버 함수를 언급할 떄마다 그 포인터가 쓰인다.
- `Widget`의 임의의 멤버 함수에서 컴파일러는 내부적으로 `divisor`를 `this->divisor`로 대체한다.
- 갈무리하려는 변수의 **지역 복사본**을 만들어서 그 복사본을 갈무리하면 해결할 수 있다.
```cpp
void Widget::addFilter() const
{
  auto divisorCopy = divisor;              // 자료 멤버를 복사한다.
  filters.emplace_back(
    [divisorCopy](int value)               // 복사본을 갈무리한다.
    { return value % divisorCopy == 0; }   // 복사본을 사용한다.
  );
}
```
- 이 접근 방식을 사용하면 기본 값 갈무리 모드도 잘 동작한다.
```cpp
void Widget::addFilter() const
{
  auto divisorCopy = divisor;              // 자료 멤버를 복사한다.
  filters.emplace_back(
    [=](int value)                         // 복사본을 갈무리한다.
    { return value % divisorCopy == 0; }   // 복사본을 사용한다.
  );
}
```
- `C++14`에서는 [일반화된 람다 갈무리](/chapter6/Item32.md)를 사용하여 **멤버 변수**를 갈무리 할 수 있다.
```cpp
// C++14
void Widget::addFilter() const
{
  filters.emplace_back(
    [divisor = divisor](int value)        // divisor를 클로저에 복사한다.
    { return value % divisorCopy == 0; }  // 복사본을 사용한다.
  );
}
```
- 일반화된 람다 갈무리에는 기본 갈무리 모드라는 것이 없으므로 `C++14`에서도 기본 갈무리 모드를 피할 수 있다.

### static과 값에 의한 갈무리 모드의 오해
- 값에 의한 갈무리 모드는 해당 클로져가 독립적이고 외부 데이터의 변화로부터 격리되어 있다는 오해를 부를 수 있다.
- 람다는 **지역 변수**와 **매개변수** 뿐만 아니라 **정적 저장소 수명 기간을 가진 객체**에도 의존할 수 있다.
  - 전역 범위나 이름 공간 범위에서 정의된 객체와 클래스, 함수, 파일 안에서 `static`으로 선언된 객체가 해당한다.
  - 그런 객체를 람다 안에서 사용할 수는 있지만, 갈무리할 수는 없다.
```cpp
void addDivisorFilter()
{
    static auto calc1 = computeSomeValue1();            // 정적 변수
    static auto calc2 = computeSomeValue2();            // 정적 변수

    static auto divisor = computeDivisor(calc1, calc2); // 정적 변수

    filters.emplace_back(
        [=](int value)                                  // 아무것도 갈무리 하지 않음
        { return value % divisor == 0; }                // 위의 정적 변수를 지칭한다.
    );

    ++divisor;                                          // divisor를 수정한다.
}
```
 - `=`를 보고 람다식에서 사용하는 `divisor`는 복사본이라고 생각할 수도 있다.
   - 이 람다는 그 어떤 비정적 지역 변수도 사용하지 않으므로 아무것도 갈무리 하지 않는다.
 - 람다의 코드는 `static`변수 `divisor`를 지칭한다.
 - `addDivisorFilter`의 각 호출의 끝에서 `divisor`가 증가하며 이 함수를 통해서 `filters`에 추가된 람다는 이전과는 다른 행동을 보이게 된다.
 - 이 람다는 `divisor`를 **참조로 갈무리**한 것과 같으며 이는 기본 **값 갈무리 모드**가 뜻하는 바와 직접적으로 모순이 된다.
   - 기본 값 갈무리 모드를 사용하지 않는다면 오해의 여지가 큰 코드가 만들어진 위험도 사라진다.