# 항목 26. 보편 참조에 대한 중복적재를 피하라
## 문제의 도입
- 사람 이름 하나를 매개변수로 받고, 현재 날짜와 시간을 기록하고 이름을 전역 자료구조에 추가하는 함수를 작성하면 아래와 같이 될것이다.
```cpp
std::multiset<std::string> names; // 전역 자료구조

template<typename T>
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_clock::now(); // 현재 시간을 얻는다.
    log(now, "logAndAdd");                       // 로그에 기록한다.
    names.emplace(std:;forward<T>(name));        // 이름을 전역 자료구조에 추가한다.
}
```
- 비합리적인 코드라고 할 수는 없지만, **비효율성**이 많이 남아있는 코드이다.
  - 이 함수를 아래 세 가지 방식으로 호출한다.

### 왼값 std::string을 넘겨줄 경우
```cpp
std::string petName("Darla");
logAndAdd(petName); // 왼값 std::string을 넘겨줌
```
- 매개변수 `name`은 변수 `petName`에 묶인다.
- `name`은 `names.emplace`으로 전달된다.
- `name`이 왼값이므로 `emplace`는 `names`에 **복사**한다.

### 오른값 std::string을 넘겨줄 경우
```cpp
logAndAdd(std::string("Persephone"));
```
- 매개변수 `name`이 오른값(`Persephone`로부터 **명시적**으로 생성되는 임시 `std::string`객체)에 묶인다.
- `name` 자체는 왼값이므로 `names`에 **복사**된다.
- `name`의 값을 `names`로 **이동**하는것이 가능하다.
- 복사 1회의 비용이 발생하나 복사를 피하고 한 번의 이동으로 작업을 완수할 여지가 있다.

### 문자열 리터럴을 넘겨줄 경우
```cpp
logAndAdd("Patty Dog");                // 문자열 리터럴을 넘겨줌
```
- 매개변수 `name`이 오른값(`Patty Dog`로부터 **암시적**으로 생성된 임시 `std::string` 객체)에 묶인다.
- `name`은 `names`에 **복사**되나 `logAndAdd`에 전달된 인수가 **문자열 리터럴**이다.
- 문자열 리터럴을 `emplace`에 직접 전달했다면 `emplace`는 `std::multiset`안에서 직접 `std::string` 객체를 생성했을 것이다.
  - 임시 `std::string`객체를 생성할 필요가 없다.
- 복사 1회의 비용을 치르지만 원칙적으로 복사는 및 이동 비용을 치를 필요도 없다.

## 비효율성 제거
- `logAndAdd`가 [보편 참조](/Chapter5/Item24.md)를 받게 하고 [참조를 std::forward를 이용해 emplace에 전달](/Chapter5/Item25.md)하면 **둘째 호출**과 **셋째 호출**의 **비효율성을 제거**할 수 있다.

### 개선한 logAndAdd 함수
```cpp
template<typename T>
void logAnddAdd(T&& name)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

std::string petName("Darla");
logAnddAdd(petName);                  // 이전처럼 왼값이 multiset으로 복사된다.
logAndAdd(std::string("Persephone")); // 오른값을 이동한다.
logAndAdd("Patty Dog");               // 임시 std::string객체를 복사하는 대신, multiset 안에 std::string을 생성한다.
```
- 가장 **최적화된 코드**가 만들어지게 된다!

## 또 다른 경우의 등장
- 이름이 아니라 인덱스를 통해 이름을 받아와서 그걸 전역 자료구조에 추가해야할 경우에는 어떻게 처리할까 ?  
  - 가장 간단한 해결책은 **함수 오버로딩**일 것이다.

### logAndAdd를 오버로딩한 버전
```cpp
std::string nameFromIdx(int idx); // idx에 해당하는 이름을 돌려준다

void logAndAdd(int idx)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(nameFromIdx(idx));
}
```
- 이 함수는 `int`를 전달할때는 문제없이 동작한다.
- `int`가 아닌 `short`를 전달하게 되면 어떻게 될까 ?

```cpp
short nameIdx;
...                 // nameIdx에 값을 배정
logAndAdd(nameIdx); // 오류!
```
- 마지막 줄에서 에러가 발생한다.

### 오류 발생 이유
- 보편 참조를 받는 버전에서는 `short`를 `short&` 로 연역할 수 있으며 주어진 인수와 **정확히 부합**한다.
- `int` 를 받는 버전에서는 `short`를 `int`로 **승격**해야 호출과 부합한다.
- 오버로딩 해소 규칙에 따르면 **정확한 부합**이 **승격보다 우선시**되기 때문에 **보편 참조를 사용하는 함수가 호출**되게 된다. 

### 보편 참조를 사용하는 함수를 호출하게 된 결과
- 보편 참조를 받는 함수에서 매개변수 `name`은 `short`에 묶인다.
- `name`이 `std::forward`를 통해서 `names`의 멤버 함수 `emplace`에 전달된다.
- `emplace`는 `std::string` 생성자에 전달한다.
- `std::string`의 생성자 중 `short`를 받는 버전은 없으므로 **생성자 호출이 실패**한다.
- 오버로딩을 시도한 개발자가 예상한 것보다 훨씬 많은 인수 형식들을 빨아들인다는 점에서 보편 참조와 오버로딩을 결합하는 것은 안좋은 선택이다.

## 완벽 전달 생성자
- 문제를 피하는 방법 중 하나는 **완벽 전달 생성자**를 작성하는 것이다.
- `std::string` 또는 색인을 받는 자유 함수를 작성하는 대신 그와 같은 일을 하는 생성자들을 가진 `Person` 클래스를 도입한다면 ?
```cpp
class Person {
public:
    template<typename T>
    explicit Person(T&& n)          // 완벽 전달 생성자
    : name(std::forward<T>(n)) {}   // 자료 멤버를 초기화한다.
    
    explicit Person(int idx)
    : name(nameFromIdx(idx)) {}     // int를 받는 생성자
    
private:
    std::string name;
};
```
- `int` 이외의 정수 형식을 넘겨주면 **보편 참조**를 받는 **생성자가 호출**되고 **컴파일에 실패**한다.
- `Person`에는 보기보다 더 많은 오버로딩 버전들이 존재한다.
  - 자동으로 **복사 생성자**와 **이동 생성자**가 생성되기 때문이다.
- `Person`은 사실상 아래와 같은 모습이 된다.
```cpp
class Person {
public:
    template<typename T>           // 완벽 전달 생성자
    explicit Person(T&& n)
    : name(std::forward<T>(n)) {}
    
    explicit Person(int idx);      // int를 받는 생성자
    
    Person(const Person& rhs);     // 복사 생성자(컴파일러 자동 생성)
    
    Person(Person&& rhs);          // 이동 생성자(컴파일러 자동 생성)
    ...
};

Person p("Nancy");
auto cloneOfPerson(P);  // p로부터 새 Person을 생성. 컴파일에 실패한다.
```
- `Person` 객체로부터 새로운 `Person`객체를 생성하려 한다.
  - 복사 생성자를 호출하는 **복사 생성의 전형적인 예**처럼 보인다.
- 실제로는 **완벽 전달 생성자를 호출**한다.
- `Person`의 `std::string` 멤버를 `Person` 객체로 생성하려 하는데 `std::string`에는 `Person`을 받는 생성자가 없다.
  - 컴파일러는 유효한 목적 코드의 산출을 포기한다.

### 컴파일러의 추론
- `cloneOfPerson`는 `const`가 아닌 **왼값**(p)으로 초기화된다.
- 템플릿화된 생성자를 `Person` 형식의 `비const` 왼값을 받는 형태로 인스턴스화 할 수 있다.
- 인스턴스화가 끝나면 `Person` 클래스는 다음과 같은 모습이 된다.
```cpp
class Person {
public:
    explicit Person(Person& n)           // 완벽 전달 템플릿에서
    : name(std::forward<Person&>(n)) {}  // 인스턴스화됨
    
    explicit Person(int idx);            // 이전과 동일
    
    Person(const Person& rhs);           // 복사 생성자 (컴파일러가 생성함)
    ...
};

auto cloneOfPerson(p);
```
- `p`는 **복사 생성자에 전달**될 수도 있고 **인스턴스화된 템플릿에 전달**될 수도 있다.
- 복사 생성자를 호출하려면 `p`에 `const`를 추가하여 인수를 **복사생성자의 매개변수 형식**(const Person&)과 부합시켜야 한다.
- 인스턴스화된 템플릿은 이미 **정확히 부합**한다.
- 컴파일러는 오버로딩 해소 규칙에 따라 더 잘 부합하는 버전을 호출하는 코드를 산출한다.
  - **템플릿에서 인스턴스화된 오버로딩**이 더 나은 부합이다.
  - `Person` 형식의 `비const` 왼값의 **복사**를 **완벽 전달 생성자가 처리**한다.

### 복사 생성자가 호출되도록 하는 방법
- 복사할 객체가 `const`가 되도록 수정하면 상황이 달라진다.
```cpp
const Person cp("Nancy");
auto clone0fP(cp);        // 복사 생성자를 호출한다.

// Person의 인스턴스화
class Person{
public:
  explicit Person(const Person& n); // 템플릿에서 인스턴스화 됨
  Person(const Person& rhs);        // 복사 생성자 (컴파일러가 생성함)
}
```
- 복사할 객체가 `const`이므로 복사 생성자가 받는 매개변수와 정확히 부합한다.
- 템플릿화된 생성자도 복사 생성자와 동일한 서명으로 인스턴스화 될 수 있다.
- 오버로딩 해소 규칙 중에는 함수 호출이 **템플릿 인스턴스**와 **일반 함수**에 똑같이 부합한다면 **일반 함수를 우선시** 한다는 규칙이 있다.
  - **복사 생성자**가 호출 대상으로 선택된다.

## 상속이 관여하는 클래스의 경우
- 상속이 관여하는 클래스에서는 **완벽 전달 생성자**와 **컴파일러**가 작성한 복사 및 이동 연산들 사이의 상호작용이 미치는 여파가 더욱 커진다.
- 파생 클래스에서 통상적인 방식으로 구현한 복사 및 이동 연산들이 놀라운 행동을 보인다.
```cpp
class SpecialPerson: public person {
public:
    SpecialPerson(const SpecialPerson& rhs) // 복사 생성자
    : Person(rhs)                           // 기반 클래스의 완벽
    { ... }                                 // 전달 생성자를 호출
    
    SpecialPeron(SpecialPerson&& rhs)       // 이동 생성자
    : Person(std::move(rhs))                // 기반 클래스의 완벽
    { ... }                                 // 전달 생성자를 호출
};
```
- 파생 클래스의 복사 및 이동 생성자들은 부모 클래스의 복사 및 이동 생성자들을 호출하는 것이 아니다.
  - **부모 클래스의 완벽 전달 생성자**를 호출한다.
- `SpecialPerson`을 받는 `std::string` 생성자가 없어서 코드가 컴파일 되지 않는다.