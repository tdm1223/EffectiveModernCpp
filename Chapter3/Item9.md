# 항목 9. typedef보다 별칭 선언을 선호하라
## typedef와 별칭 선언
```cpp
typedef std::unique_ptr<std::unordered_map<std::string, std::string>> UPtrMapSS;
using UPtrMapSS = std::unique_ptr<std::unordered_map<std::string, std::string>>;
```
- `typedef`와 별칭 선언이 하는 일은 정확히 동일하다.
- `typedef`는 템플릿화 할 수 없지만 별칭 선언은 템플릿화 할 수 있다.
- `typedef`보다 별칭 선언을 선호해야하는 이유는 별칭 템플릿(alias templates)때문이다. 

## 별칭 템플릿
```cpp
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

template<typename T>
struct MyAllocList {
    typedef std::list<T, MyAlloc<T>> type;
};
```
- 별칭 선언을 사용한 경우가 간단하다.

## 좀 더 나쁜 상황
- 템플릿 매개변수로 지정된 형식의 객체들을 담는 연결 목록을 생성하려는 목적으로 템플릿 안에서 `typedef`를 사용하려 한다면 `typedef` 이름 앞에 `typename`을 붙여야 한다.
```cpp
template<typename T>
class Widget{
private:
    typename MyAllocList<T>::type list;
};
```
- `MyAllocList<T>::type`은 템플릿 형식 매개변수(T)에 의존적인 형식을 지칭한다.
- `MyAllocList<T>::type`은 **의존적 형식**이며 `C++`은 의존적 형식의 이름 앞에 반드시 `typename`을 붙여주어야 한다.

```cpp
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

template<typename T>
class Widget {
private:
    MyAllocList<T> list;
};
```
- 컴파일러가 `Widget` 템플릿을 처리하는 과정에서 `MyAllocList<T>`가 쓰인 부분에 도달했을때 컴파일러는 `MyAllocList<t>`가 **형식의 이름**임을 알고 있다.
  - `MyAllocList`가 **형식 템플릿**이므로, `MyAllocList<T>`는 반드시 형식의 이름이어야 하기 때문이다.
- `MyAllocList<T>`는 비의존적 형식이며 `typename` 지정자를 붙일 필요가 없고 붙여서도 안된다.
- `::type` 접미어도 붙일 필요가 없다.

## typename이 문제를 일으키는 경우
```cpp
class Wine { ... };

template<>
class MyAllocList<Wine> {
private:
    enum class WineType
    { White, Red, Rose };
    
    WineType type; // 이 클래스에서 type은 멤버변수 이름이 된다
    ...
};
```
- 위와 같이 선언하는 경우 `MyAllocList<Wine>::type`은 형식을 지칭하지 않는다.
- `Wine`으로 `Widget`을 인스턴스화하면 `Widget` 템플릿 안의 `MyAllocList<T>::type`은 형식이 아니라 자료 멤버를 지칭한다.
- `Widget` 템플릿 안에서 `MyAllocList<T>::type`이 형식을 지칭하는지는 전적으로 T가 무엇인지에 의존한다.
- 컴파일러가 안심하고 받아들이게 하기 위해서는 `typename`을 붙여야 한다.

## STL에서 type을 수정한 사례
```cpp
std::remove_const<T>::type
std::remove_reference<T>::type
std::add_lvalue_reference<T>::type
```
- `STL`에서 `C++11` 이전에는 `type`을 사용하였다.
- 문제가 발생할 수 있어 `C++14`에서는 아래의 `STL`이 추가되었다.

```cpp
std::remove_const_t<T>
std::remove_reference_t<T>
std::add_lvalue_reference_t<T>
```