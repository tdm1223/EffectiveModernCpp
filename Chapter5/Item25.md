# 항목 25. 오른값 참조에는 std::move를, 보편 참조에는 std::forward를 사용하라
## 오른값 참조 정리
- 오른값 참조는 **이동할 수 있는 객체**에만 묶인다.
- 어떤 매개변수가 오른값 참조라면 그 참조에 묶인 객체를 **이동할 수 있음이 확실**하다.
```cpp
class Widget{
    Widget(Widget&& rhs); // rhs는 이동이 가능한 객체를 참조하는것이 확실하다.
}
```
- 객체를 다른 함수에 넘겨주되 함수가 객체의 **오른값 성질을 활용**할 수 있도록 넘겨주어야 하는 경우도 생긴다.
  - 객체에 묶이는 매개변수를 **오른값으로 캐스팅**해야 한다.
  - [std::move가 하는일](/Chapter5/Item23.md)이 그것이며 `std::move`의 존재 이유이다.

```cpp
class Widget
{
public:
    Widget(Widget&& rhs)
    : name(std::move(rhs.name)),
      p(std::move(rhs.p))
    { ... }
    ...

private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};
```
- 오른값 침조를 다른 함수로 전달할 때는 `std::move`를 통해 **오른값으로의 무조건 캐스팅**을 적용해야 한다.
  - 그런 참조는 **항상** 오른값에 묶이기 때문이다.
- [오른값 참조에 std::forward를 사용해도 원하는 행동이 일어나게 하는것이 가능](/Chapter5/Item23.md)하지만 소스 코드가 길어지고 실수의 여지가 있으며 관용구에서 벗어난 모습이 된다.
  - 오른값 참조에서 `std::forward`를 사용하는 것은 피해야 한다.

## 보편 참조 정리
- [보편 참조는 이동에 적합한 객체에 묶일 수도 있고 아닐 수도 있다.](/Chapter5/Item24.md)
- 보편 참조는 오른값으로 초기화되는 경우에만 오른값으로 캐스팅되어야 한다.
- [std::forward가 하는일](/Chapter5/Item23.md)이 그것이다.

```cpp
class Widget
{
public:
    template<typename T>
    void setName(T&& newName)
    { name = std::forward<T>(newName); } // newName은 보편참조
    ...
};
```
- 보편 참조를 다른 함수도 전달할 때에는 `std::forward`를 통해 오른값으로의 **조건부 캐스팅**을 적용해야 한다.
  - 그런 참조는 **특정 조건하에서만** 오른값에 묶이기 때문이다.
- 보편 참조에 `std::move`를 사용하는것은 매우 위험 하다.
  - 왼값(예를들면 지역변수)이 의도치 않게 수정되는 결과가 발생할 수 있기 때문이다.

## 보편 참조에 std::move를 사용하면 위험한 이유
```cpp
class Widget {
public:
    template<typename T>
    void setName(T&& newName)          // 보편 참조
    { name = std::move(newName); }     // 컴파일되긴 하지만 아주 나쁘다 !
    ...
    
private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};

std::string getWidgetName();  // 팩토리 함수

Widget w;

auto n = getWidgetName();     // n은 지역 변수

w.setName(n);                 // n을 w로 이동한다. 이제 n의 값은 알 수 없다.
...
```
- `setName` 함수에 지역변수 `n`을 전달해준다. 
- `n`을 읽기 전용으로 사용할 것이라 생각하겠지만 내부적으로 `std::move`를 이용하여 참조 인수를 오른값으로 무조건 캐스팅한다.
- `setName` 함수가 끝난 이후에는 `n`은 **미지정 값**이 된다.
- `setName`의 매개변수를 보편 참조로 선언한 것 자체가 문제라고 생각할 수 있다.
  - `setName`은 자신의 매개변수를 수정하지 말아야 하므로 `const`를 명시하는것이 바람직 하지만, [보편 참조는 const일 수 없다.](/Chapter5/Item24.md)
- 문제를 해결하기 위해 매개변수를 보편참조로 선언하지 않고 함수 오버로딩을 이용해서 해결하자는 생각을 할 수도 있다.

### 오버로딩을 적용한 코드
```cpp
class Widget {
public:
    void setName(const std::string& newName) // const 왼값으로 name을 설정
    { name = snewName; }
    
    void setName(std::string&& newNmae)      // 오른값으로 name을 설정
    { name = std::move(newName); }
    
    ...
};
```
- 함수 오버로딩이 하나의 해결책일 수는 있지만 몇 가지 단점이 있다.
#### 1. 작성하고 유지보수해야 할 소스 코드의 양이 늘어난다.
#### 2. 효율성이 떨어질 수 있다.
```cpp
class Widget {
public:
    ...  
    
    void setName(std::string&& newNmae)  // 임시 객체이므로 오른값
    { name = std::move(newName); }       // name으로 이동 후 임시 객체 소멸
    
    ...
};

...

w.setName("Adela Novak");  // 문자열 리터럴 전달 및 임시 객체 생성
```
- **문자열 리터럴**을 파라미터로 전달할 경우 보편 참조를 사용했더라면 문자열 리터럴이 `name`에 대한 할당 연산자의 인수로 쓰였을 것이다. 
  - 문자열 리터럴이 `w`의 멤버 `name`에 직접 할당되며 임시 `std::string`객체는 생성되지 않는다.
- 오버로딩 버전에서는 문자열 리터럴로부터 임시 `std::string` 객체가 생성되어 `setName`의 매개변수에 묶이고 임시 `std::string`이 `w`의 멤버로 이동한다.
  - `setName`을 한 번 호출했을 때 아래 3개 함수가 한 번 실행될 수 있다. 
    - `std::string` **생성자**(임시 객체 생성을 위해) 한 번
    - **이동 할당 연산자**(`newName`을 `w.name`으로 이동하기 위해) 한 번
    - **소멸자**(임시 객체를 파괴하기 위해) 한번

#### 3. 코드양이 기하급수적으로 증가한다.
- `setName`은 매개변수를 하나만 받으므로 오버로딩이 두개면 되었다.
- 매개변수가 더 많고 각 매개변수가 **왼값**일 수도 있고 **오른값**일 수도 있다면 오버로딩의 수가 기하급수적으로 증가한다.
  - 매개변수가 `n`개이면 필요한 오버로딩 버전의 수는 2<sup>n</sup>이다.
- 함수 템플릿들 중에는 **왼값**일 수도 있고 **오른값**일 수도 있는 매개변수들을 **무제한으로 받는 것들이 존재**한다.
  - 대표적으로 `std::make_shared`와 `std::make_unique`가 있다.
  - 이런 함수들은 왼값과 오른값에 대한 오버로딩이 불가능하고 유일한 방안은 보편 참조이다.
  - 이런 함수의 내부에서 보편 참조를 전달할 때는 `std::forward`를 사용해야 한다. 

## 함수 내부에서 참조를 여러번 사용할 경우
- 보편 참조이든 오른값 참조이든 함수 내부적으로 여러번 사용하는 경우가 있다.
- 객체를 다 사용하기 전에 다른 객체로 이동하는 일은 피해야 한다.
- 이러한 경우에는 반드시 마지막에만 `std::move`(오른값 참조의 경우) 또는 `std::forward`(보편 참조의 경우)를 사용해야 한다.
```cpp
template<typename T>
void setSignText(T&& text)                       // text는 보편 참조
{
    sign.setText(text);                          // text를 사용하되 수정하지는 않는다.
    auto now = std::chrono::system_clock::now(); // 형재 시간을 얻는다.
    signHistory.add(now, std::forward<T>(text)); // text를 오른값으로 조건부 캐스팅
}
```
- `sign.setText` 함수에서 `text`의 값을 읽기만 해야 하고 변경하지 않아야 한다.
  - `signHistory.add`를 호출할 때 `text`의 값을 사용하기 때문이다.
- 보편 참조를 마지막으로 사용하는 지점에서만 적용한 것은 바로 이때문이다.
- `std::move`에 대해서도 같은 논리가 적용된다.
  - `std::move` 대신 [std::move_if_noexcept를 사용하는 게 바람직한 경우](/Chapter3/Item14.md)도 있다.

## 반환에서의 이동
- 함수가 결과를 **값으로 반환**하고 오른값 참조나 보편 참조에 붂인 객체라면 반환문에서 `std::move`나 `std::forward`를 사용하는 것이 바람직하다.

### 오른값 참조에서 call by value
- 두 행렬을 더하는, 좌변의 행렬이 오른값임이 알려진 `operator+` 함수
```cpp
Matrix
operator+(Matrix&& lhs, const Matrix& rhs) // 결과를 값으로 반환
{
    lhs += rhs;
    return std::move(lhs);
}
```
- `return` 문에서 `lhs`를 **오른값으로 캐스팅**한 덕분에 컴파일러는 `lhs`를 함수의 **반환값 장소로 이동**한다.
```cpp
Matrix
operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return lhs; // lhs를 반환값으로 복사한다.
}
```
- `std::move` 호출이 없다면 `lhs`가 **왼값**이므로 컴파일러는 **반환값 장소로 복사**해야 한다.
- `Matrix` 형식이 복사 생성보다 효율적인 이동 생성을 지원한다고 할 때 `return` 문에서 `std::move`를 사용하면 좀 더 효율적인 코드가 만들어진다.
- `Matrix`가 이동을 지원하지 않는다면 ?
  - 오른값으로의 캐스팅이 문제가 되지 않는다.
  - 오른값이 `Matrix`의 [복사 생성자에 의해 복사](/Chapter5/Item23.md)되기 때문이다.
  - 나중에 `Matrix`의 이동 지원이 추가 되면 `operator+`를 재컴파일했을 때 **효율성이 저절로 향상**된다.

### 보편 참조에서 call by value
- 보편 참조에서도 그대로 적용이 되어 원래의 객체가 오른값이라면 객체의 값을 반환값으로 이동하는 것이 바람직하다.
```cpp
template<typename T>
Fraction reduceAndCopy(T&& frac) // 값 전달 방실의 반환값 보편 참조 매개변수를 받는다
{
    frac.reduce();
    return std::forward<T>(frac); // 오른값은 반환값으로 이동하고 왼값은 복사한다
}
```

## 반환에서의 이동을 잘못 적용할 경우
- 함수가 반환할 **지역 변수에도 최적화를 적용**할 수 있을 것이라고 생각할 수 있다.
- 아래와 같이 **지역변수를 값으로 반환**하는 함수가 있다.
```cpp
Widget makeWidget()      // makeWidget의 복사 버전
{
    Widget w;            // 지역 변수
    ...                  // w를 설정한다.
    return w;            // w를 반환값에 복사한다.
}
```
- 복사하는 부분을 아래처럼 이동으로 바꿈으로써 함수를 최적화할 수 있다고 생각할 수 있다.
```cpp
Widget makeWidget()      // makeWidget의 이동 버전
{
    Widget w;
    ...
    return std::move(w); // w를 반환값으로 이동한다.
}
```
- **복사**를 **이동**으로 바꾸었으니 성능이 향상될 것이라고 생각할 것이다.
- 복사 버전에서 만일 지역 변수 `w`를 함수의 반환값을 위해 마련한 메모리 안에 생성한다면 `w`의 복사를 피할수 있다.
  - 반환값 최적화 `RVO(return value optimization)`가 일어나게 된다.

### RVO가 일어나기 위한 조건
1. **지역 객체의 형식**이 **함수의 반환 형식**과 같다.
2. **지역 객체**가 **함수의 반환값**이어야 한다.

### 복사 버전과 이동 버전에 대한 설명
- 복사버전에서 두 조건을 만족하므로 제대로된 컴파일러라면 반드시 반환값 최적화를 적용해서 w의 복사를 피할것이다.
- 이동버전처럼 `std::move`를 사용하게 되면 반환하는 것이 **지역 객체w**가 아닌 `std::move(w)`의 결과인 **w의 참조**가 된다.
  - 지역 객체에 대한 참조를 돌려주는 것은 반환값 최적화의 필수 조건을 만족하지 못한다.
  - 컴파일러는 반드시 `w`를 함수의 **반환값 장소**로 옮겨야 한다.
  - 컴파일러의 최적화를 도우려했지만 컴파일러가 할 수 있는 최적화 여지를 제한하게 된다.
- 함수가 지역 객체를 값으로 반환하는 경우 지역 객체에 `std::move`를 적용한다고 해서 컴파일러에게 도움이 되지는 않는다.