# 항목 27. 보편 참조에 대한 중복적재 대신 사용할 수 있는 기법들을 알아 두라
## 중복적재 포기
- `logAndAdd` 오버로딩 버전을 작성하지 않는 것이다.
  - 인자로 보편 참조를 받는 경우 함수명을 `logAndAddName`으로 한다.
  - 인자로 `index`를 받는 경우 함수명을 `logAndAddNameIdx`로 한다.
- 오버로딩이 없기 때문에 문제가 발생하지 않는다.
- 클래스 생성자 관련 문제(`Person` 예제)에서는 사용할 수 없다는 단점이 있다.
  - **생성자의 이름은 고정**되기 때문이다.

## constT& 매개변수를 사용
- `C++98`로 돌아가서 보편 참조 매개변수 대신 `const`에 대한 **왼값 참조 매개변수**를 사용한다.
- [logAndAdd](/Chapter5/Item26.md)의 첫번째 버전이 사용한 접근 방식이다.
  - 효율성의 문제가 발생할 수 있으나 예상치 않은 문제를 회피할 수 있다.

## 값 전달 방식의 매개변수 사용
- 복잡도를 높이지 않고 성능을 높일 수 있는 한 가지 접근방식은 **참조 전달 매개변수** 대신 **값 전달 매개변수**를 사용하는 것이다.
```cpp
class Person {
public:
  explicit Person(std::string n) // T&& 생성자를 대체한다
  : name(std::move(n)) {}
    
  explicit Person(int idx)
  : name(nameFromIdx(idx)) {}
  ...
    
private:
  std::string name;
};
```
- `std::string` 관련 타입은 모두 **첫 번째 생성자**를 호출한다.
- `int` 관련 타입(std::size_t, short, long 등)은 모두 **두 번째 생성자**를 호출한다.

## 꼬리표 배분 사용
- `const` 왼값 참조 전달이나 값 전달은 **완벽 전달**을 지원하지 않는다.
- 완벽 전달을 위하여 보편 참조를 사용하려는 것이라면 보편 참조 외에는 대안이 존재하지 않는다.
- 보편 참조와 오버로딩 둘 다 사용하되 보편 참조에 대한 오버로딩을 피하려면 **꼬리표 배분방식**을 사용할 수 있다.
```cpp
std::multiset<std::string> names;

template<typename T>
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
```
- `int`형식의 인덱스를 받는 오버로딩을 추가하면 [골치아픈 문제들](/Chapter5/Item26.md)이 발생한다.
- 오버로딩 대신 `logAndAdd`가 호출을 **다른 함수로 위임**하게 한다.
  - `logAndAdd` 함수는 **인터페이스** 역할을 하고 `logAndAddImpl` 함수를 추가하여 실제 동작을 하도록 한다.
  - 이들에 대해서는 오버로딩을 사용한다.

### 꼬리표 배분을 적용한 logAndAdd
```cpp
template<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(std::forward<T>(name), std::is_integral<T>()); // 아주 정확하지는 않음
}
```
- `logAndAdd` 자체는 모든 인수 형식(정수 형식과 비정수 형식)을 받는다.
- 자신의 매개변수를 `logAndAddImpl`에 전달하며 매개변수의 형식이 **정수인지를 뜻하는 인수**도 전달한다.
- 오른값인 정수 인수들에 대해서는 원하는 방식으로 작동한다.
- 보편 참조 `name`으로 전달된 인수가 **왼값**이면 `T`는 **왼값 참조 형식으로 연역**된다.
  - `int` 형식의 왼값이 `logAndAdd`에 전달되면 `T`는 `int&`로 연역된다.
  - 참조는 정수 형식이 아니므로 실제로 정수 값을 나타내도 `std::is_integral<T>()`는 왼값 인수에 대해 거짓이 된다.

### 꼬리표 배분을 적용한 logAndAdd의 문제 해결 방법
- [std::remove_reference](/Chapter3/Item9.md)라는 형식에 **참조 한정사를 제거**하는 형식 특질을 사용하면 된다.
```cpp
//C++ 11
template<typename T>
void logAndAdd(T&& name)
{
  logAndAddImpl(std::forward<T>(name), std::is_integral<typename std::remove_reference<T>::type>());
}

//C++14
typename<typename T>
void logAndAdd(T&& name)
{
  logAndAddImpl(std::forward<T>(name), std::is_integral<std::remove_reference_t<T>>());
}
```

### logAndAddImpl의 구현
- `logAndAdd`는 자신에게 전달된 인수가 정수 형식인지 아닌지 뜻하는 `bool`값을 `logAndAddImpl`에 넘겨준다.
- `true`와 `false`는 **실행 시점** 값이다.
- 현재 하려고 하는 것은 **컴파일 시점**의 현상인 오버로딩 해소 과정에서 적절한 함수가 선택되게 하는 것이다.
- 필요한 것은 `true`, `false`에 해당하는 어떤 형식이다.
- 표준 라이브러리는 `std::true_type`과 `std::false_type`이라는 형식들을 제공한다.

#### 정수가 아닌 형식에만 적용 되는 버전
```cpp
template<typename T>
void logAndAddImpl(T&& name, std::false_type)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
```
#### T가 정수 형식인 경우
```cpp
std::string nameFromIdx(int idx);

void logAndAddImpl(int idx, std::true_type)
{
    logAndAdd(nameFromIdx(idx));
}
```
- 주어진 인덱스로 해당 이름을 조회해서 `logAndAdd`를 호출한다.
- 이름은 `std:forward`를 통해서 또 다른 `logAndAddImpl` 오버로딩 버전으로 전달된다.

### 꼬리표 배분 정리
- `std::trye_type` 형식과 `std::false_type` 형식은 오버로딩 해소가 원하는 방식으로 일어나게 하는 데에만 쓰이는 일종의 **꼬리표**이다.
  - 매개변수들에 이름을 붙이지도 않았다.
- `logAndAdd`안에서 오버로딩된 구현 함수들을 호출하는 것은 주어진 작업을 꼬리표에 기초해서 **배분**하는 것에 해당한다.
- 이러한 설계를 꼬리표 배분이라고 부른다.

## 보편 참조를 받는 템플릿 제한
- 꼬리표 배분의 필수 요소는 클라이언트 `API` 역할을 하는 오버로딩되지 않은 함수이다.
  - 그 함수는 요청된 작업을 구현 함수들로 배분한다.
- 오버로딩 되지 않은 배분 함수를 작성하는 것은 어렵지 않으나 [완벽 전달 생성자에 관련된 문제점](/Chapter5/Item26.md)은 예외이다.
- 컴파일러가 **자동으로 복사 생성자와 이동 생성자를 작성**할 수 있다.
- 생성자를 하나만 작성해서 꼬리표 배분을 사용한다면 일부 생성자 호출을 **컴파일러가 작성한 함수들**로 처리되어서 꼬리표 배분이 적용되지 않을 위험이 있다.
  - 컴파일러가 작성한 함수들이 꼬리표 배분 설계를 **항상 우회하지는 않는다**는 점이 큰 문제이다.
- 보편 참조 매개변수를 포함하는 함수 템플릿이 오버로딩 해소의 후보가 되는 조건들을 적절히 제한할 수 있는 기법이 필요하다.

### std::enable_if를 활용한 해결
- 컴파일러가 **특정 템플릿이 존재하지 않는것**처럼 행동할 수 있다.
  - `std::enable_if`가 적용된 템플릿을 **비활성화된 템플릿**이라 부른다.
- 기본적으로 모든 템플릿은 **활성화된 상태**이나 `std::enable_if`를 사용하는 템플릿은 `std::enable_if`에 지정된 조건이 만족될 때에만 활성화된다.
- `Person` 예제의 경우 **Person이 아닌 형식의 인수가 전달된 경우**에만 `Person`의 완벽전달 생성자가 활성화되게 해야 한다.
- `Person` 형식이 전달되었다면 **완벽 전달 생성자를 비활성화**해야한다.
  - 해당 호출은 복사 생성자나 이동 생성자가 처리하게 될것이다.
```cpp
class Person {
public:
    template<typename T, typename = typename std::enable_if<조건>::type>
    explicit Person(T&& n);

    ...
};
```
- `T`가 `Person`이 아닐 때에만 컴파일러가 이 템플릿화된 생성자의 인스턴스를 만들어내게 해야 한다.

### std::is_same 사용
- 조건을 지정하고자 할 때 유용한 형식 특질로 두 형식이 같은지를 판정하는 `std::is_same`이 있다.
- 왼값으로 초기화되는 보편 참조는 항상 왼값으로 연역되기 때문에 정확한 해답은 아니다.
  - 보편 생성자의 형식 `T`는 `Person&`로 연역된다.
  - `Person`이라는 형식과 `Person&`라는 형식은 같지 않으므로 `std::is_same`은 둘이 다르다고 판정한다.
  - 결과적으로 `std::is_same<Person, Person&>::value`는 거짓이 된다.

### std::decay를 사용하여 const, volatile, 참조여부 해결
- `Person`의 템플릿화된 생성자가 오직 `T`가 `Person`이 **아닐 때에만 활성화**되어야 한다는 조건은 형식 `T`에 대해 아래 두 사항을 무시해야 한다는점과 같다.
  - 참조여부 : 보편 참조 생성자의 활성화 여부를 판정할 때 `Person`, `Person&`, `Person&&`은 모두 `Person`과 같은 형식으로 간주되어야 한다.
  - `const`성과 `volatile`성 : `const Person`, `volatile Person`, `const volatile Person`은 모두 `Person`과 같은 형식으로 간주되어야 한다.
- `T`가 `Person`과 같은지 판정하기 전에 `T`에서 모든 참조 한정사와 `const`, `volatile`을 제거하는 수단이 필요하다.
- 이와 같은 수식어들을 없애는 방법은 `std::decay` 함수를 이용하는 것이다.
  - `std::decay<T>::type`은 `T`에서 **모든 참조**와 모든 **cv-한정사**를 제거한 형식에 해당한다.
- 생성자의 활성화를 제어하는 바람직한 조건은 아래와 같다.
  - `!std::is_same<Person, typename std::decay<T>::type>::value`
```cpp
class Person {
public:
  template<
    typename T,
    typename = typename std::enable_if<
                 !std::is_same<Person,
                                typename std::decay<T>::type
                              >::value
               >::type
    >
    explicit Person(T&& n);
    ...
};
```
- `Person`을 다른 어떤 `Person`으로 생성하는 경우 **오른값**이든 **왼값**이든 **const유무**, **volatile유무**에 상관없이 **보편 참조**를 받는 생성자는 절대 호출되지 않는다.

### 파생 클래스 문제 해결
- 아직 [파생 클래스에서의 문제들](/Chapter5/Item26.md)에 대해서 해결하지 못하였다.
  - 파생 클래스의 경우에도 인스턴스를 생성하지 못하도록 해야한다.
- 주어진 형식이 다른 한 형식에서 **파생된 것인지 알려주는 형식 특질**이 표준 라이브러리에 존재한다.
  - `std::is_base_of<T1, T2>::value`는 `T2`가 `T1`에서 파생된 형식이면 참이다.
  - 사용자 정의 형식은 자기 자신으로부터 파생된 것으로 간주된다.
  - `std::base_of<T, T>::value`는 `T`가 사용자 정의 형식이라면 참이된다.
- 위 코드에서 `std::is_same` 대신에 `std::is_base_of`를 사용하면 된다.
```cpp
class Person {
public:
  template<
    typename T,
    typename = typename std::enable_if<
                 !std::is_base_of<Person,
                                  typename std::decay<T>::type
                                 >::value
               >::type
    >
    explicit Person(T&& n);
    ...
};
```

- `C++14`에서는 `std::enable_if`와 `std::decay`의 별칭 템플릿을 이용해서 `typename`과 `::type`을 제거할 수 있다.
```cpp
class Person
{
public:
  template<
    typename T,
    typename = std::enable_if_t<
                !std::is_base_of<Person,
                                 std::decay_t<T>
                                >::value
               >
  >
  explicit Person(T&& n);
};
```

### 정수 타입의 문제 해결
- 정수 타입의 문제는 `is_integral`함수를 사용하면 된다.
```cpp
class Person
{
public:
  template<
    typename T,
    typename = std::enable_if_t<
      !std::is_base_of<Person, std::decay_t<T>>::value
      &&
      !std::is_integral<std::remove_reference_t<T>>::value
    >
  >
  
  explicit Person(T&& n)            // std::string이나
  : name(std::forward<T>(n))        // std::string으로 변환되는
  { ... }                           // 인수를 위한 생성자 
    
  explicit Person(int idx)          // 정수 인수를 위한 생성자
  : name(nameFromIdx(idx)) 
  { ... }

  ...                               // 기타 복사, 이동 연산자들
};
```

## 절충점들
- 오버로딩 포기, `const T&` 전달, 값을 전달하는 것은 호출되는 함수들의 **각 매개변수에 대해 형식을 지정**한다.
- 꼬리표 배분과 템플릿 활성화 제한은 **완벽 전달을 사용**하므로 매개변수들의 형식을 지정하지 않는다.
- 완벽 전달이 더 효율적인 것은 사실이지만 문제점을 갖고 있다.

### 완벽 전달의 문제점
#### [완벽 전달이 불가능한 인수들](/Chapter5/Item30.md)이 있다.
- 구체적인 형식을 받는 함수에는 전달할 수 있어도 **완벽 전달은 불가능** 할 수 있다.

#### 클라이언트가 유효하지 않은 인수를 전달했을 때 나오는 오류 메시지가 난해하다.
```cpp
Person p(u"Konrad Zuse");
```
- 16비트 문자로 구성된 문자열을 넘겼을 경우 `char16_t[12]`를 `std::string`이나 `int`로 변환할 수 없다는 에러 메시지가 뜬다.
- 완벽 전달에 기초한 접근 방식의 경우 컴파일러가 `const char16_t`들의 배열을 아무 불평없이 생성자의 매개변수에 묶는다.
  - 생성자는 매개변수를 `Person`의 `std::string` 생성자에 전달하며 호출자가 전달한 것과 필요한 것 사이의 불일치가 발견된다.
  - 이때 나타나는 에러 메시지는 굉장히 길고 복잡해서 어떤 에러가 발생했는지 쉽게 알아내기 힘들다
- `Person`의 경우 전달 함수의 보편 참조 매개변수가 `std::string`에 대한 초기치로 쓰일것을 알고있다.
  - 초기치로 사용하는 것이 가능한지를 미리 `static_assert`를 이용해서 **점검하는 방법**도 있다.
```cpp
class Person {
public:
  template<
    typename T,
    typename = std::enable_if<
      !std::is_base_of<Person, std::decay_t<T>>::value
      &&
      !std::is_Integral<std::remove_reference_t<T>>::value
    >
  >
  
  explicit Person(T&& n)
  : name(std:;forward<T>(n))
  {
    // T 객체로부터 std::string을 생성할 수 있는지 점검한다.
    static_assert(
      std::is_constructible<std::string, T>::value,
      "Parameter n can't be used to construct a std::string"
   );
   
    ... // 일반적인 생성자 작업을 수행한다.
  }

  ... // Person 클래스이 나머지 부분(이전과 동일)
private:
    std::string name;
};
```
- `std::is_constructible`이라는 형식 특질은 한 형식의 객체를 다른 한 형식의 객체로부터 생성할 수 있는지를 컴파일 시점에서 판정한다.
- 복잡한 에러 메시지들 대신에 깔끔하고 명확한 에러 메시지를 보여줄 수 있다.
- 매개변수를 전달하는 코드는 멤버 초기화 목록에 있기 때문에 `static_assert`보다 먼저 호출된다.
  - 복잡한 에러 메시지가 나오고 나서 맨 마지막에서야 `static_assert`로 인한 에러 메시지가 나온다.