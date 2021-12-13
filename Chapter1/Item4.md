# 항목 4. 연역된 형식을 파악하는 방법을 알아두라
 - 형식 연역 결과를 직접 확인하는 수단은 소프트웨어 개발 과정에서 정보가 필요한 시점에 따라 다르다.
 - 코드를 작성 및 수정하는 시점, 컴파일 시점, 실행시점 세가지 시점에서 형식 연역 정보를 얻는 방법을 살펴본다.

## IDE 편집기
- `visual studio`같은 코드 편집기는 마우스 커서를 올리면 개체의 형식을 표시해 준다.

![123](/Img/vs.jpg)

## 컴파일러의 진단 메시지
- 원하는 형식 때문에 컴파일에 문제를 발생하게 하는 방법을 통해 컴파일러가 연역한 형식을 파악할 수 있다.
- `decltype`과 `template`를 사용한다.

```cpp
template<typename T> // TD를 선언만 해둔다.
class TD;

const int answer = 10;
auto x = answer;
auto y = &answer;

TD<decltype(x)> xType;  // 컴파일 에러
TD<decltype(y)> yType;  // 컴파일 에러
```
- `xType`, `yType`을 선언한 부분에서 컴파일 에러가 나고 `x`와 `y`의 타입이 무엇인지 알 수 있다.

![type](/Img/typeerror.jpg)

## 실행시점 출력
### typeid와 std::type_info::name
```cpp
std::cout << typeid(x).name() << std::endl;
std::cout << typeid(y).name() << std::endl;
```
- x와 y에 대해 연역된 형식이 출력된다.

```cpp
template<typename T>
void f(const T& param)
{
  std::cout << typeid(T).name() << std::endl; // T를 표시
  std::cout << typeid(param).name() << std::endl; // param의 형식을 표시
}

std::vector<Widget> createVec(); // 팩토리 함수
const auto vw = createVec(); // vw를 팩토리 함수의 반환값으로 초기화
f(&vw[0]); //  호출
```
- `T` 와 `param` 모두 `const Widget*`로 연역된다.(GNU, Clang, Microsoft 컴파일러)
  - `param` 과 `T` 의 타입이 같은게 말이 되지 않는다.
  - `T`가 `int`였다면, `param`의 형식은 `const int&`가 되어야 한다.

- `param`의 형식을 일부러 틀린다. 표준에 따르면 `std::type_info::name` 은 **값 전달 매개변수**로 취급해야하기 때문이다.
  - [값 전달의 경우 형식이 참조이면 참조성이 무시되며, 참조를 제거한 후의 형식이 const이면 const역시 무시된다.](/Chapter1/Item1.md)
- 실제 `param`의 형식은 `const Widget * const&` 인데 **값 전달**이기 때문에 뒤에 나타난 `const` 와 참조가 사라진다.

### boost 라이브러리의 typeindex
```cpp
template<typename T>
void f(const T& param)
{
  using boost::typeindex::type_id_with_cvr;
  std::cout << type_id_with_cvr<T>().pretty_name() << std::endl;
  std::cout << type_id_with_cvr<decltype(param)>().pretty_name() << std::endl;
}
```
- 함수 템플릿 `boost::typeindex::type_id_with_cvr`은 자신에게 전달된 형식 인수의 `const`, `volatile`, 참조 한정사들을 그대로 보존한다.
- `boost::typeindex::type_id_with_cvr`은 하나의 `boost::typeindex::type_index` 객체를 산출한다.
- `pretty_name` 멤버 함수는 형식을 사람이 보기 좋게 표현한 문자열을 담은 `std::string` 객체를 반환한다.
