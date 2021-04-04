# 항목 7. 객체 생성 시 괄호와 중괄호를 구분하라
## C++ 객체 생성 구문
```cpp
int x(0); // 초기치를 괄호로 감싼 예
int x = 0; // 초기치를 "=" 다음에 지정한 예
int x{0}; // 초기치를 중괄호로 감싼 예
int x = {0}; // "=-"와 중괄호로 초기치를 지정한 예
```
- 4가지 방법 모두 똑같은 결과를 나타낸다.
- 중괄호`{}`로 나타낸 생성 방법을 **균일 초기화**이라고 한다.

## 균일 초기화 (uniform initialization)
- 중괄호를 사용하는 초기화이다.
- C++이 지원하는 세 가지 초기화 표현식 지정 방법 중 어디서나 사용할 수 있기에 균일 초기화라고 한다.

### 컨테이너의 초기 내용을 초기화
```cpp
std::vector<int> v{ 1, 3, 5};
```
### 비정적 자료 멤버의 기본 초기화 값을 지정
```cpp
class Widget {
   private:
   int x{ 0 }; // 가능
   int y = 0;  // 가능
   int z(0);   // 오류
}
```
### 복사할 수 없는 객체를 초기화

```cpp
std::atomic<int> a{ 0 };  // 가능
std::atomic<int> a(0);    // 가능
std::atomic<int> a = 0;   // 오류
```

## 균일 초기화의 장점
### 좁히기 변환(narrow conversion) 방지
```cpp
double x, y, z;

int sum1{ x + y + z}; // 오류 double들의 합을 int로 표현하지 못할 수 있다.
int sum2( x + y + z); // 표현식의 값이 int에 맞게 잘려나간다.
int sum3 = x + y + z; // 표현식의 값이 int에 맞게 잘려나간다.
```

### 가장 성가신 구문해석(most vexing parse)에 자유로움
- C++규칙중엔 `"선언으로 해석할 수 있는 것은 항상 선언으로 해석해야 한다."`라는 것이 있다.
  - 이 규칙에 의해 기본 생성자를 호출하려고 할때 컴파일러가 함수명인지 변수명인지 구분하지 못하는 것이다.
- 균일 초기화를 통해 인수 없는 생성자를 호출할 수 있다.

```cpp
Widget w(10); // 인수 10으로 Widget의 생성자 호출
Widget w1();  // w1이라는 함수 선언 (가장 성가신 구문해석)
Widget w2{};  // 인수 없는 Widget의 생성자 호출
```

## 균일 초기화의 단점
### std::initializer_list 문제
- 생성자 중에 `std::initializer_list`를 파라미터로 사용하는 경우 예상치 못한 문제가 발생한다.
- [항목2](/Chapter1/Item2.md)에서 처럼 다른 방식으로 선언된 변수에 대해서는 균일 초기화가 좀 더 직관적인 형식으로 연역되었지만 auto로 선언된 변수에 대해서는 `std::initializer_list` 형식으로 연역되는 경우가 많다.
  - `auto`를 많이 사용할 수록 균일 초기화를 멀리하는 경향이 생기게 된다.

```cpp
class Widget {
public:
    Widget(int i, bool b);
    Widget(int i, double d);
    Widget(std::initializer_list<long double> il);
    operator float() const; // float로 변환
}

Widget w1(10, true);    // Widget(int i, bool b) 호출
Widget w2{10, true};    // Widget(std::initializer_list<long double> il) 호출
Widget w3(10, 5.0);     // Widget(int i, double d) 호출
Widget w4{10, 5.0};     // Widget(std::initializer_list<long double> il) 호출

Widget w5(w4);          // 괄호 사용, 복사 생성자 호출
Widget w6{w4};          // 중괄호 사용, std::initializer_list 생성자 호출

Widget w7(std::move(w4)); // 괄호 사용, 이동 생성자 호출
Widget w8{std::move(w4)}; // 중괄호 사용, std::initizlier_list 생성자 호출
```
- 생성자 이외에도 복사생성이나 이동 생성이 일어났을 상황에서도 `std::initializer_list` 생성자가 끼어든다.
- 컴파일러가 보통의 중복적재 해소로 물러나는 경우도 존재한다.
  - 중괄호 초기치의 인수 형식들을 `std::initializer_list` 안의 형식으로 변환하는 방법이 아예 없을때이다.
```cpp
class Widget {
public:
    Widget(int i, bool b);
    Widget(int i, double d);
    Widget(std::initializer_list<std::string> il); // std::initizlier_list의 원소 형식이 std::string
}

Widget w1(10, true);    // Widget(int i, bool b) 호출
Widget w1{10, true};    // Widget(int i, bool b) 호출
Widget w1(10, 5.0);     // Widget(int i, double d) 호출
Widget w1{10, 5.0};     // Widget(int i, double d) 호출
```
- 위 코드처럼 좁히기 변환이 불가능한 경우(int -> string)에 균일 초기화를 사용할 수 있다.

### container 초기화
- `std::initializer_list`는 컨테이너를 초기화하는데 사용한다.
- 빈 컨테이너를 만들고 싶다면 어떻게 해야할까?
    - 빈 중괄호 쌍을 괄호로 감싸거나 빈 중괄호 쌍을 또 다른 중괄호 쌍으로 감싸면 된다.
```cpp
Widget w1({}); // std::initializer_list 생성자를 빈 초기치 목록으로 호출
Widget w2{{}}; // std::initializer_list 생성자를 빈 초기치 목록으로 호출
```

## 중괄호 초기화? 괄호 초기화?
### 중괄호 초기화를 기본으로 사용
- 다양한 문맥에 적용 가능
- 좁히기 변환 방지
- 가장 성가신 구문해석에서 자유로움
- 괄호가 필요한 경우(주어진 크기와 초기 요소 값으로 `std::vector`를 생성하는 경우 등)에는 괄호 사용

### 괄호 초기화를 기본으로 사용
- C++ 구문적 전통과의 일관성
- `auto`가 `std::initializer_list`를 연역하는데 문제가 없음
- 객체 생성 시 의도치 않게 `std::initializer_list` 생성자가 호출되는 일이 없다는 점
- 중괄호가 필요한 경우(구체적인 값들로 컨테이너를 생성 하는 경우 등)에는 중괄호 사용