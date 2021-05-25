# 항목 41. 이동이 저렴하고 항상 복사되는 복사 가능 매개변수에 대해서는 값 전달을 고려하라
## 중복 함수에 대한 처리
- 매개변수 중에는 애초에 복사되도록 만들어진 것들이 있다.
  
### 오버로딩
```cpp
class Widget{
public:
  void addName(const std::string& newName)  // 왼값을 받아서
  { names.push_back(newName); }             // 복사한다.

  void addName(std::string&& newName)       // 오른값을 받아서
  { names.push_back(std::move(newName)); }  // 이동한다.
private:
  std::vector<std::string> names;
};
```
- `addName`의 효율성을 위해서는 **왼값 인수는 복사**하되 **오른값 인수는 이동**하는 것이 바람직하다.
- 코드가 잘 작동하긴 하지만 같은 일을 하는 함수를 두 개 작성해야 한다.
  - 목적 코드에도 함수가 두 개 존재하게 된다.
- 한가지 대안은 `addName`함수를 [보편 참조를 받는 함수 템플릿](/Chapter5/Item24.md)으로 만드는 것이다.

### 보편 참조
```cpp
class Widget{
public:
  template<typename T>
  void addName(T&& newName)
  {
    names.push_back(std::forward<T>(newName));
  }
};
```
- 소스 코드 길이는 줄어들지만 보편 참조를 사용하기 때문에 그와 관련된 여러 문제점들이 발생할 수 있다.
- 목적 코드에는 이 템플릿의 서로 다른 인스턴스가 여러 개 포함될 수 있다.
  - 왼값과 오른값에 대해 다르게 인스턴스화 된다.
  - `std::string`과 `std::string`으로 변환 가능한 형식들에 대해서도 [다르게 인스턴스화](/Chapter5/Item25.md) 된다.
- [보편 참조로는 전달할 수 없는 인수 형식들이 존재한다는 점](/Chapter5/Item30.md)도 문제가 된다.
- 클라이언트가 부적절한 형식의 인수를 전달하면 컴파일러가 [난해한 오류 메시지](/Chapter5/Item27.md)를 출력할 수 있다.

### 값전달
- 발생하는 문제점들을 피하는 방법 중 하나는 **사용자 정의 형식 객체는 값으로 전달하지 말라는 규칙**을 포기하면 된다.
- 값전달로 받는 코드의 모습은 아래와 같다.
```cpp
class Widget {
public:
  void addName(std::string newName)
  { names.push_back(std::move(newName)); }
  ...
};
```
- `addName`함수가 하나이므로 소스 코드와 목적 코드에서 코드 중복이 없다는 점은 명백하다.
- 템플릿과 보편 참조를 사용하지 않으므로 헤더 파일의 크기가 커지는 문제나 전달 못하는 인수 형식이 존재하는 문제도 없다.
- 난해한 오류 메시지가 나오는 문제도 없다.
- 그러나 이전부터 값 전달의 성능은 좋지 않다고 알려져왔다.
- 실제로 `C++98` 에서는 비용이 컸다. 
  - `C++98`에서는 호출자가 무엇을 넘겨주든 매개변수 `newName`이 **복사 생성**에 의해 생성된다.
- `C++11`에서 `newName`은 인수가 **왼값일 때에만 복사 생성**되고, **오른값일 때에는 이동 생성**된다.
```cpp
Widget w;
...
std::string name("Bart");
w.addName(name);           // addName을 왼값으로 호출
...
w.addName(name + "Jenne"); // addName을 오른값으로 호출
```
- 첫 번째 `addName` 호출에서 매개변수 `newName`은 왼값으로 초기화 된다.
  - `newName`은 `C++98`처럼 **복사 생성**된다.
- 두 번째 `addName` 호출에서 매개변수 `newName`은 `std::string`에 대한 `operator+` 호출로부터 생성되는 `std::string` 객체로 초기화된다.
  - 그 객체는 오른값이므로 `newName`은 **이동 생성**된다.

## 성능 비교
- 이름 하나를 `Widget`에추가하는 작업의 비용을 복사 및 이동 연산 기준으로 확인해본다.

### 오버 로딩
```cpp
class Widget {
public:
  void addName(const std::string& newName)
  { names.push_back(newName); }
  
  void addName(std::string&& newName)
  { names.push_back(std::move(newName); }
  
  ...
  
private:
  std::vector<std::string> names;
};
```
- 호출자가 넘겨준 인수가 왼값이든 오른값이든 그 인수는 `newName`이라는 참조에 묶인다.
- 복사, 이동 연산을 기준으로 한 비용은 없다.
- 함수 본문에서는 왼값 오버로딩에서 `newName`이 `Widget::names`로 복사되고 오른값 오버로딩에서는 이동된다.
- 왼값의 경우 복사 1회, 오른값의 경우 이동 1회의 비용이 든다.

### 보편 참조
```cpp
class Widget {
public:
  template<typename T>
  void addName(T&& newName)
  { names.push_back(std::forward<T>(newName)); }
  
  ...
private:
  std::vector<std::string> names;
};
```
- 오버로딩 구현에서처럼 호출자의 인수는 `newName`에 묶인다.
  - 비용이 없는 연산이다.
- 함수 본문에서는 `std::forward` 때문에 왼값 `std::string` 인수는 `Widget::names`에 복사되고 오른값 `std::string` 인수는 이동된다.
- 왼값의 경우 복사 1회, 오른값의 경우 이동 1회의 비용이 든다.

### 값 전달
```cpp
class Widget {
public:
  void addName(std::string newName)
  { names.push_back(std::move(newName)); }
  
  ...
private:
  std::vector<std::string> names;
};
```
- 호출자가 넘겨준 인수가 왼값이든 오른값이든 매개변수 `newName`이 **반드시 생성**된다.
- 인수가 왼값이면 비용은 복사 생성 1회이고 오른값이면 이동 생성 1회이다.
- 함수 본문에서는 어떤 경우이든 `newName`이 항상 `Widget::names`로 이동된다.
- 왼값의 경우 복사 1회 + 이동 1회, 오른값의 경우 이동 2회의 비용이 든다.

## 주목해야될 네 가지 사항
### 값 전달을 사용하라가 아니라 고려하라이다.
- 값 전달 방식에는 함수를 하나만 작성하면 된다는 장점과 목적 코드에 함수 하나만 만들어진다는 장점, 보편 참조와 관련된 문제점이 없다는 장점이 있다.
- 다른 대안들보다 비용이 크며 추가 비용도 존재한다.

### 복사 가능 매개변수에 대해서만 값 전달을 고려해야 한다.
- 복사할 수 없는 매개변수는 이동 전용 형식일 것이다.
  - 함수가 복사 가능이 아닌 매개변수의 복사본을 항상 만들어 낸다면 그 복사본은 반드시 이동 생성자를 통해 생성되는 것이기 때문이다.
- 값 전달 방식의 장점은 함수를 하나만 작성하면 된다는 것이다.
- 이동 전용 형식에서는 왼값 인수를 위한 오버로딩을 따로 둘 필요가 없다.
  - 왼값의 복사는 복사 생성자의 호출로 이어지는데 이동 전용 형식에서는 복사 생성자가 비활성화되어 있기 때문이다.
- 따라서 오른값 인수를 위한 오버로딩만 제공하면된다.
  - 이 경우 오버로딩의 해소에는 오버로딩 버전 하나(오른값 참조를 받는 버전)만 있으면 된다.

### 값 전달은 이동이 저렴한 매개변수에 대해서만 고려해야 한다.
- 이동이 저렴한 경우에는 이동이 한 번 더 일어나도 큰 문제가 되지 않는다.
- 이동의 비용이 크다면, 불필요한 이동을 수행하는 것은 불필요한 복사를 수행하는 것과 비슷하다.
  - 불필요한 복사 연산을 피하는 것은 중요한 문제이다.

### 값 전달은 항상 복사되는 매개변수에 대해서만 고려해야 한다.
- 아무것도 추가되지 않는 경우에도 생성과 파괴 비용을 유발하는 경우가 존재한다.
- 참조 전달 접근 방식들에서는 치를 필요가 없는 비용이다.

## 매개변수를 복사하는 방식
- 이동이 저렴한 복사 가능 형식에 대해 항상 복사를 수행하는 함수라고 해도 값 전달이 적합하지 않은 경우가 있다.
- 함수가 매개변수를 복사하는 방식이 두 가지이기 때문이다.

### 생성을 통한 복사
- `addName`은 **생성**을 사용한다.
- 자신의 매개변수 `newName`을 `vector::push_back`에 넘겨주며 그 함수는 복사 생성을 이용해서 `newName`을 `std::vector`의 끝에 생성된 새 요소로 복사한다.
- 생성을 통해서 매개변수를 복사하는 함수의 비용은 값 전달의 비용과 같다.
  - 다른 대안 들에 비해 이동이 한 번 더 수행된다.

### 배정을 통한 복사
- 매개변수를 배정을 통해서 복사하는 함수에서는 상황이 복잡하다.

```cpp
class Password {
public:
  explicit Password(std::string pwd)
  : text(std::move(pwd)) {}
  
  void changeTo(std::string newPwd)
  { text = std::move(newPwd); }
  
  ...
  
private:
  std::string text;
};
```
- 패스워드를 나타내는 클래스로 값 전달 방식으로 구현한 예이다.
- 패스워드는 바뀔수 있으므로 `changeTo`라는 설정 함수를 제공한다.

```cpp
std::string initPwd("Supercalifragilisticexpialidocious");
Password p(initPwd);

std::string newPassword = "Beware the Jabberwock";
p.changeTo(newPassword);
```
- `changeTo`가 매개변수 `newPassword`를 복사하려면 배정을 수행해야 하며 함수의 **값 전달 접근방식** 때문에 비용이 아주 커질 수 있다.
- `changeTo`에 전달된 인수는 **왼값**이다.
  - 매개변수 `newPassword`가 생성될 때 호출되는 것은 `std::string`의 **복사 생성자**이다.
  - 생성자는 새로 담을 메모리를 할당한다.
  - `newPassword`가 `text`로 이동 배정되는데 이에 의해 `text`가 차지하고 있던 메모리가 해제된다.
  - `changeTo`안에서 동적 메모리 관리 동작이 두 번(새 패스워드를 담을 메모리 할당, 기존 메모리가 차지하던 메모리 해제) 일어난다.
- 기존 패스워드가 새 패스워드보다 길기 때문에 메모리를 할당하거나 해제할 필요가 없다.
  - 오버로딩 접근 방식을 사용한다면 할당과 해제를 생략할 수 있다.
```cpp
class Password {
public:
  ...
  
  void changeTo(const std::string& newPwd)
  {
    text = newPwd;
  }
  
  ...
private:
  std::string text;
};
```
- 기존 패스워드가 새것보다 짧다면 대체로 배정 도중 할당-해제 쌍을 피하는것이 불가능하다.
  - 그런 경우 값 전달이 참조 전달과 거의 비슷한 빠르기로 실행될 것이다.
- 매개변수를 배정 연산을 통해서 복사하는 함수에서 값 전달의 이러한 추가 비용은 여러 상황에 의존한다.
  - 전달되는 형식, 왼값 인수 대 오른값 인수 비율, 형식이 동적 메모리 할당을 사용하는지의 여부
- 동적 메모리 할당을 사용한다면 그 형식의 배정 연산자의 구현 방식과 배정 대상에 연관된 메모리가 배정의 원본에 연관된 메모리만큼 큰지의 여부에도 의존한다.
- `std::string`의 경우에는 구현이 [작은 문자열 최적화](/Chapter5/Item29.md)를 사용하는지, 사용한다면 배정되는 값이 `SSO` 버퍼에 들어가는 크기인지에도 의존한다.

## 잘림 문제
- 참조 전달과는 달리 값 전달에서는 잘림 문제가 발생할 여지가 있다.
- 함수가 기반 클래스 형식이나 그로부터 파생된 임의의 형식의 매개변수를 받는 경우에는 그 매개변수를 값 전달 방식으로 선언하지 않는 것이 좋다.
  - 그런 매개변수를 값 전달로 선언하면 파생 형식의 객체가 전달되었을 때 그 객체의 파생 클래스 부분이 잘려 나가기 때문이다.

```cpp
class Widget{...};                         // 기본 클래스

class SpecialWidget: public Widget{ ... }; // 파생 클래스

void processWidget(Widget w);              // Widget과 그로부터 파생된 임의의 형식을 받는 함수. 잘림 문제가 있다.

SpecialWidget sw;

proceswsWidget(sw);                        // processWidget은 이를 SpecialWidget이 아니라 Widget으로 인식한다.
```
