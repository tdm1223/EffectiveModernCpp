# 항목 21. new를 직접 사용하는 것보다 std::make_unique와 std::make_shared를 선호하라
## std::make_unqiue와 std::make_shared
- `std::make_shared`는 `C++11` 부터 지원하지만 `std::make_unique`는` C++14`에서 표준 라이브러에 포함되었다.
  - `C++11`에서는 `std::make_unique`의 기본적인 버전을 직접 작성한다.
```cpp
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
    return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```
- `make_unique`는 자신의 매개변수들을 생성할 객체의 생성자로 전달한다.
- `new`가 돌려준 `raw` 포인터로 `std::unique_ptr`를 생성해서 돌려준다.

## std::make_unique와 std::make_shared 사용의 장점
### 간결한 코드
- 스마트 포인터를 `make` 함수들을 이용해서 생성하는 것과 그냥 생성하는 것만 봐도 `make` 함수들을 선호할 이유가 명확하다.
```cpp
auto upw1(std::make_unique<Widget>());      // make 함수를 사용
std::unique_ptr<Widget> upw2(new Widget()); // 사용하지 않음

auto spw1(std::make_shared<Widget>());      // make 함수를 사용
std::shared_ptr<Widget> spw2(new Widget()); // 사용하지 않음
```
- 타입 선언을 한 번만 작성하는데 이는 소프트웨어 공학의 핵심 교의 중 하나인 **코드 중복을 피하라**를 잘 지키는 것이다.
- 소스 코드의 중복이 있으면 컴파일 시간이 늘어나고 목적 코드의 덩치가 커지고 코드 기반을 다루기가 좀 더 어려워 진다.
- 중복된 코드는 일관성이 없는 코드로 진화하기 쉽고 코드 기반의 비일관성은 버그로 이어지는 경우가 많다.

### 예외 안정성
```cpp
void processWidget(std::shared_ptr<Widget> spw, int priority); // Widget을 객체의 우선순위에 따라 적절히 처리하는 함수
int computePriority(); // 우선순위를 계산하는 함수

processWidget(std::shared_ptr<Widget>(new Widget), computePriority()); // new를 사용한 함수에서 우선순위 계산 사용
```
- 함수를 호출하는 코드를 세분화하면 3단계로 나뉜다.
1. `new Widget`
   - 표현식 `new Widget`이 평가된다.
   - `Widget`이 **힙**에 생성된다.
2. `std::shared_ptr<Widget>()`
   - `new`가 산출한 포인터를 관리하는 `std::shared_ptr<Widget>`의 생성자가 실행된다.
3. `computePriority()`
   - 우선순위 계산 함수가 실행된다.
- `Widget`의 생성자와 `std::shared_ptr`의 생성자는 순서대로 진행된다.
- 실행 시점에서 `std::make_shared`가 먼저 호출될 수도 있고 `computePriority`가 먼저 호출될 수도 있다.
  - `computePriority()` 함수가 먼저 호출될 수 있는데 `computePriority()` 함수에서 예외가 발생하면 `Widget` 객체는 누수가 발생하게 된다.

```cpp
processWidget(std::make_shared<Widgget>(), computePriority()); // 자원 누수의 위험이 없음
```
- `std::make_shared`가 먼저 호출되면 동적으로 할당된 `Widget`을 가리키는 `raw` 포인터는 `computePriority()`가 호출되기 전에 반환된 `std::shared_ptr`안에 안전하게 저장된다.
  - `computePrioirity()`가 예외를 방출하면 `std::shared_ptr`의 소멸자가 피지칭 `Widget` 객체를 파괴한다.
- `computePrioirity()`가 먼저 호출되고 예외를 방출한다면 `std::make_shared`는 호출되지 않는다.
  - `Widget`이 동적으로 할당되지 않는다.
  - 누수를 걱정할 자원이 아예 없다.
- `std::make_shared`를 사용하면 예외가 발생했더라도 `Widget`객체는 `std::shared_ptr`로 관리되어 누수가 발생하지 않았을 것이다.

### 메모리 할당의 효율성
```cpp
std::shared_ptr<Widget> spw(new Widget);
```
- `new`를 사용하여 생성하면 **두 번의 할당**이 일어난다.
  - 객체 생성
  - 제어블록 생성

```cpp
auto spw = std::make_shared<Widget>();
```
- `std::make_shared`는 **Widget 객체**와 **제어 블록** 모두를 담을 수 있는 크기의 메모리 조각을 한 번에 할당한다.
  - 메모리 할당 호출 코드가 한 번만 있으면 되므로 프로그램의 **정적 크기**가 줄어든다.
  - 제어 블록에 내부 관리용 정보를 포함할 필요가 없어져서 프로그램의 전체적인 메모리 사용량이 줄어들 여지가 생긴다.

## make 함수들을 사용하지 못하거나 사용하지 않아야 하는 상황들
### 커스텀 삭제자
- `make` 함수들 중에는 **커스텀 삭제자**를 지정할 수 있는 것이 없다.
- 커스텀 삭제자를 사용하는 스마트 포인터를 `new`를 이용해서 생성하는 것은 간단하다.
```cpp
auto widgetDeleter = [](Widget* pw){ ... }; //  커스텀 삭제자
std::unique_ptr<Widget, decltype(widgetDeleter)> upw(new Widget, widgetDeleter);
std::shared_ptr<Widget> spw(new Widget, widgetDeleter);
```

### std::initializer_list
```cpp
auto upv = std::make_unique<std::vector<int>>(10, 20);
auto spv = std::make_shared<std::vector<int>>(10, 20);
```
- 위 호출을 실행했을때 스마트 포인터가 가리키는 값은 ?
  - 값이 20인 요소 10개를 담은 벡터
  - 값이 각각 10과 20인 두 요소를 담은 벡터
- 두 호출 모두 **값이 20인 요소 10개를 담은 벡터**를 생성한다.
  - `make` 함수들이 내부적으로 매개변수들을 전달할 때 **중괄호가 아니라 괄호를 사용**함을 뜻한다.
- 값이 각각 10과 20인 두 요소를 담은 벡터를 생성하려면 ?
  - `auto` 형식 연역을 이용해서 중괄호 초기치로부터 `std::initializer_list` 객체를 생성하고 `make`함수에 넘겨준다.

```cpp
// std::initializer_list 객체를 생성
auto initList = { 10, 20 };

// std::initializer_list 객체를 이용해서 std::vector를 생성
auto spv = std::make_shared<std::vector<int>>(initList);
```

### operator new와 operator deltete
- 특정 클래스들은 `new`, `delete` 연산자를 오버로딩하는 경우가 있다.
  - 전역 메모리 할당 루틴과 해제 루틴이 그 형식의 객체에 적합하지 않음을 뜻한다.
  - 이 경우 클래스의 객체와 정확히 같은 크기의 메모리를 할당하고 해제하는 경우가 많다.
- `std::shared_ptr`에서 제공하는 **커스텀 할당** 및 **삭제자**에 어울리지 않는다.
  - `std::allocate_shared`는 해당 객체 크기에 더해 **제어 블록**만큼의 크기를 더 필요로 하기 때문이다.

## 제어 블록의 weak count
- `make` 함수는 메모리를 한 번에 할당한다.
  - 메모리 해제도 한 번에 해야한다.
- 제어 블록에는 제어 블록을 참조하는 `std::weak_ptr`들의 개수에 해당하는 또 다른 참조 횟수도 있다.
  - 둘째 참조를 `weak count`라고 부른다.
- 제어 블록을 참조하는 `std::weak_ptr`가 존재하는 한(`weak count`가 0보다 크다면) 해제할 수 없다. 
  - 제어 블록을 담고 있는 메모리도 여전히 할당된 상태여야 한다.
- `new` 로 `std::shared_ptr`를 생성했다면 **제어 블록도 따로 생성**되기 때문에 `std::weak_ptr`가 존재하여도 소멸될 수 있다.
- 메모리 크기가 매우 큰 객체를 생성한다면 `make`를 사용하지 않는 것이 메모리 관리에 유리하다.