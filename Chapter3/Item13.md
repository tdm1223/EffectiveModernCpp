# 항목 13. iterator보다 const_iterator를 선호하라
### const_iterator
- const를 가리키는 포인터의 STL 버전이다.
- 수정하면 안되는 값들을 가리킨다.
- 반복자가 가리키는 것을 수정할 필요가 없을 때에는 항상 `const_iterator`를 사용하는 것이 바람직하다.
  - `const_iterator` 는 `iterator`와 달리 수정하면 안 되는 값들을 가리킨다.

```cpp
std::vector<int> values;
...
std::vector<int>::iterator it = std::find(values.begin(), values.end(), 1983);
values.insert(it, 1998);
```
- 1983 이라는 값을 찾아 그 위치에 1998 이라는 값을 삽입하는 코드이다.
- `find`함수로 찾은 `iterator`를 직접 수정하는 일은 없다.
  - `iterator`가 최선의 선택이 아니다.
  - 표준 관행에 따르도록 하면 `iterator`를 `const_iterator`로 바꿔야 한다.

### C++98
- `C++98` 에서는 `const_iterator`를 얻을 방법이 없었기 때문에 쉽지가 않았다.
- `const_iterator`를 얻으려면 아래과 같이 작성해야한다.
```cpp
typedef std::vector<int>::iterator IterT;
typedef std::vector<int>::const_iterator ConstIterT;

std::vector<int> values;
...
ConstIterT ci =
    std::find(static_cast<ConstIterT>(values.begin()), // 캐스팅
              static_cast<ConstIterT>(values.end()),   // 캐스팅
              1983);
values.insert(static_cast<IterT>(ci), 1998); // 컴파일이 안될 수 있음
```
- `find`호출에서 `const_iterator`를 얻기 위해서는 정적 캐스팅을 사용하였다.
  - `values`가 `비const` 컨테이너이며 `C++98`에는 `비const`컨테이너로부터 `const_iterator`를 얻는 간단한 방법이 없었기 때문이다.
- `C++98`에서는 삽입, 삭제 위치를 `iterator`로만 지정할 수 있다.
  - `const_iterator`를 허용하지 않아 다시 `iterator`로 캐스팅을 해야만 한다. (`insert`시 `iterator`로 캐스팅)
  - `const_iterator`에서 `iterator`로 캐스팅을 허용하지 않기 때문에 캐스팅한 코드도 컴파일조차 되지 않는 경우가 있다.

### C++11
- `C++11`에서는 간단해졌다.
- `cbegin()`, `cend()`함수는 `const_iterator`를 반환해주며 `insert`와 `end`는 `const_iterator`를 사용한다.

```cpp
std::vector<int> values;
...
auto it = std::find(values.cbegin(), values.cend(), 1983); // cbegin, cend 사용
values.insert(it, 1998);
```
- `generic`라이브러리 코드를 작성하는 곳에서는 `C++11`에서도 부족함이 발견된다.
  - 일반성을 극대화한 코드는 특정 멤버 함수의 존재를 요구하는 대신 그 멤버 함수에 상응하는 비멤버 함수를 사용한다.
```cpp
template<typename C, typename V>
void findAndInsert(C& container, const V& targetVal, const V& insertVal)
{
    using std::cbegin;
    using std::cend;
    
    auto it = std::find(cbegin(container),
                        cend(container),
                        targetVal);
                       
    container.insert(it, insertVal);
}
```            
- `C++14` 에서는 문제없이 동작하지만 `C++11`에서는 동작하지 않는다.
- `C++11` 표준화 과정에서 비멤버 `cbegin`과 `cend`를 추가하지 않았기 때문이다.

- 작성한 비멤버 `cbegin`
```cpp
template <class C>
auto cbegin(const C& container)->decltype(std::begin(container))
{
    return std::begin(container);
}
```
- `cbegin`이 아닌 비멤버 `begin`을 사용하는 이유
  - 이 템플릿은 컨테이너 같은 자료구조를 대표하는 임의의 인수 형식 C를 받고 해당 `const` 참조 매개변수 `container`를 통해서 그 자료구조에 접근한다.
  - `const` 컨테이너에 대해 비멤버 `begin` 함수를 호출하면 `const_iterator`형식의 반복자가 반환된다.
  - 이 템플릿이 반환하는 것이 그 반복자이다.(const_iterator)