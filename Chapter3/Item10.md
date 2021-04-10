# 항목 10. 범위 없는 enum보다 범위 있는 enum을 선호하라
## 범위 없는 enum
- 일반적으로 한 중괄호 쌍 안에서 어떤 이름을 선언하면 그 이름의 가시성은 해당 중괄호 쌍이 정의하는 범위로 한정된다.
  - `C++98` 스타일의 `enum`으로 선언된 열거자들에 대해서는 이런 규칙이 적용되지 않는다.
```cpp
enum Color { black, white, red }; // black, white, red는 Color가 속한 범위에 속함
auto white = false;  // 오류! 이 범위에 white가 선언되어 있음
```
- `white`의 범위 문제로 똑같은 `white`를 선언할 수가 없다.
- 이런 종류의 `enum`을 범위 없는 `enum`이라고 한다.

## 범위 있는 enum
- C++11의 새로운 열거형이다.
- `enum class`라는 구문으로 선언한다는 점 때문에 범위 있는 `enum`을 `enum` 클래스라고 부르기도 한다.
```cpp
enum class Color { black, white, red };
auto white = false;    // OK 이 범위에 다른 white는 없음
Color c = white;       // 오류! 이 범위에 white라는 이름의 열거자가 없음
Color c = Color:white; // OK
```
- `namespace`의 오염을 줄여준다는 것만으로도 범위 있는 `enum`을 범위 없는 `enum`보다 선호할 이유가 충분하다.

## 묵시적 변환 문제
- 범위 있는 `enum`의 또 다른 강력한 장점은 열거자들에 형식이 훨씬 강력하게 적용된다는 점이다.
  - 범위 없는 `enum`의 열거자들은 암묵적으로 정수 형식으로 변환된다.

```cpp
enum Color { black, white, red}; // 범위 없는 enum
std::vector<std::size_t> primeFactors(std::size_t x); // x의 소인수들을 돌려주는 함수

Color c = red;

if(c < 14.5) { // Color를 double과 비교한다!
    auto factors = primeFactors(c); // Color의 소인수들을 계산한다?
}
```
- 범위 있는 `enum`을 쓰면 오류가 발생한다.
```cpp
enum class Color { black, white, red};
std::vector<std::size_t> primeFactors(std::size_t x); // x의 소인수들을 돌려주는 함수

Color c = Color::red;

// 오류 발생
if(c < 14.5) {
    auto factors = primeFactors(c);
}
```
![123](/Img/enumerror.jpg)

- 다른 형식으로 변환하고 싶으면 캐스팅을 사용하면 된다. (`static_cast<double>(c) < 14.5`)

## 전방 선언
- 범위 있는 `enum`은 열거자들을 지정하지 않고 열거형 이름만 미리 선언할 수 있다.
```cpp
enum Color; // 에러
enum class Color; // OK
```

## 범위 있는 enum의 기본 타입
- 범위 있는 enum의 기본 타입은 `int`이다.
```cpp
enum class Status; // 기본 타입은 int
```
- 기본 형식이 마음에 들지 않는다면 다른 형식을 명시적으로 지정하면 된다.
- `enum`을 정의할 때에도 기본 타입을 지정할 수 있다.
```cpp
enum class Status: std::uint32_t; // 기본 타입은 std::uint32_t
enum class Status: std::unint32_t { good = 0, failed = 1, incomplete = 100}; // 정의할때 기본 타입 지정
```

## 범위 없는 enum이 유용한 상황
- `C++11`의 `std::tuple`안에 있는 필드들을 지칭할때 범위 없는 `enum`은 유용하게 사용할 수 있다.
```cpp
using UserInfo = std::tuple<std::string, std::string, std::size_t>; // 사용자 이름, 이메일 주소, 평판치
UserInfo uInfo;
auto val = std::get<1>(uInfo); // 필드 1의 값을 얻는다 (필드 1이 무엇인지 주석을 확인해야 한다)
```
- 프로그래머는 필드 1이 이메일 주소라는 점을 기억하기 힘들다.
  - 범위 없는 `enum`으로 **필드 번호를 필드 이름에 연관**시키면 기억하지 않아도 된다.

```cpp
enum UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo;
auto val = std::get<uiEmail>(uInfo); // 이메일 필드의 값을 얻는 것을 명확하게 알 수 있다.
```

### 범위 있는 enum으로 작성한 버전
```cpp
enum class UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo;
auto val = std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>(uInfo);
```
- 열거자 하나를 받아서 그에 해당하는 `std::size_t` 값을 돌려주는 함수를 작성한다면 장황함이 줄어들 수 있다.
  - 그런 함수를 작성하는 것이 다소 까다롭다.
    - 어떤 종류의 `enum`에 대해서도 작동해야 하기 때문에 `constexpr`함수 템플릿이어야 한다.
    - 반환 형식도 일반화 해야 하기 때문에 `size_t`가 아닌 `enum`의 기본 형식을 반환해야 한다.
    - 절대로 예외를 던지지 않을 것을 알고 있기에 [nonexcept](/Chapter3/Item14.md)로 선언해야 한다.

### 임의의 열거자 하나를 받아 그 값을 컴파일 시점 상수로서 돌려주는 함수 템플릿의 예
```cpp
template<typename E>
constexpr auto toUType(E enumerator) noexcept
{
    return static_cast<std::underlying_type_t<E>>(enumerator);
}
auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo); // toUType을 이용해 튜플의 한 필드에 접근
```