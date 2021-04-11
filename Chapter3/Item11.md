# 항목 11. 정의되지 않은 비공개 함수보다 삭제된 함수를 선호하라
- 클라이언트가 특정 함수를 호출하지 못하게 해야하는 경우가 존재한다.
- [특수 멤버 함수들](/Chapter3/Item17.md)이라 하는 C++이 필요에 따라 **자동으로 작성하는 멤버 함수**들에게서 발생한다.

### 정의되지 않은 비공개 함수
- `C++98`의 접근 방식은 `private`으로 선언하고 정의는 하지 않는다.
  - 이렇게 하여 작성한 함수가 제목에서 지칭하는 **정의되지 않은 비공개 함수**이다.
```cpp
private:
    basic_ios(const basic_ios&);            // not defined
    basic_ios& operator=(const basic_ios&); // not defined
```
- 함수들은 `private` 섹션에 선언되어 있으므로 클라이언트가 호출할 수 없다.
- 함수들을 의도적으로 정의하지 않았기 때문에 이들에 접근할 수 있는 코드에서 호출한다고 해도 **정의가 없어서 링크에 실패**한다.

### 삭제된 함수
- `C++11`에서는 복사 생성자와 복사 배정 연산자 선언의 끝에 `=delete`를 붙이는 더 나은 방법이 존재한다.
- `delete`를 붙인 함수가 제목에서 지칭하는 **삭제된 함수**이다.

```cpp
public:
    basic_ios(const basic_ios&) = delete;
    basic_ios& operator=(const basic_ios&) = delete;
```
- 삭제된 함수는 어떤 방법으로든 사용할 수 없다.
  - 멤버 함수나 `friend` 함수에서 `basic_ios` 객체를 복사하려하면 컴파일이 실패한다.
- 삭제된 함수는 `public`으로 선언하는 것이 관례이다.

### 삭제된 함수의 장점
- 어떤 함수도 삭제할 수 있다.
- `private`은 멤버 함수에만 적용할 수 있다.

```cpp
bool isLucky(int number); // 행운의 번호인지 여부를 돌려주는 비멤버 함수

if(isLucky('a')) ...   // a가 행운의 번호인가?
if(isLucky(true))) ... // true가 행운의 번호인가?
if(isLucky(5.5)) ...   // 5.5가 행운의 번호인가?

```
- 행운의 번호가 정수라면 위 3개의 조건문은 컴파일을 실패하도록 하는것이 바람직하다.
- 배제할 형식들에 대한 함수 오버로딩을 명시적으로 삭제하는 것이다.

```cpp
bool isLucky(int number);
bool isLucky(char) = delete;
bool isLucky(double) = delete;
bool isLucky(bool) = delete;
```

### 템플릿 인스턴스화 방지
- 삭제된 함수로 수행할수 있는 또 다른 것은 원치 않는 템플릿 인스턴스화를 방지하는 것이다.

```cpp
template<typename T>
void processPointer(T* ptr);
```
- 위 템플릿이 `char*`나 `void*`에 대해서 호출을 거부하도록 하고싶다면?
  - `private` 접근 방식을 적용할 수 없다.
```cpp
class Widget{
public:
    template<typename T>
    void processPointer(T* ptr) { }
private:
    template<typename T>
    void processPointer<void>(void*); // 오류
}
```
- `delete`를 사용하면 된다.
```cpp
template<>
void processPointer<void>(void*) = delete;

template<>
void processPointer<char>(char*) = delete;
```
- 더욱 철저하게 삭제하려면 `const void*`, `const char*`, `const volatile void*`, `const volatile char*`과 다른 표준 문자 형식들인 `wchar_t`, `char16_t`, `char32_t`등의 포인터들에 대한 버전도 삭제하면 된다.
- 템플릿 특수화는 반드시 클래스 범위가 아니라 네임스페이스 범위에서 작성해야 하기 때문이다.
  - 삭제된 함수에는 다른 접근 수준을 지정할 필요가 없으므로 이런 문제가 없다.
- 멤버 함수를 클래스 바깥에서 삭제하는 것이 가능하다.