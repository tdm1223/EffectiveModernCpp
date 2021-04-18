# 항목 15. 가능하면 항상 constexpr을 사용하라
## constexpr
- `C++11`부터 지원하는 한정자이다.
- **일반화된 상수 표현식**을 사용할 수 있게 해준다.
  - 변수나 함수, 생성자 함수에 대하여 **컴파일 시점**에 평가될 수 있도록 처리할 수 있다.
- `constexpr`이 적용된 객체는 `const`이며, 그 값은 실제로 컴파일 시점에서 알려진다.

## 객체와 변수에서의 constexpr
```cpp
int sz;                                 // 비 constexpr 변수

constexpr auto arraySize1 = sz;         // 오류! sz의 값이 컴파일 도중에 알려지지 않음
std::array<int, sz> data1;              // 오류! sz의 값이 컴파일 도중에 알려지지 않음

constexpr auto arraySize2 = 10;         // 10은 상수
std::array<int, arraySize2> data2;      // OK! arraySize2는 constexpr 객체

const auto arraySize = sz;              // arraySize는 sz의 const 복사본
std::array<int, arraySize> data;        // 오류! arraySize의 값은 컴파일 시점에서 알려지지 않음

```
- 모든 `constexpr` 객체는 `const`이지만 모든 `const` 객체가 `constexpr`인 것은 아니다.
- `constexpr`이 아닌 `const`로 선언하였다면 컴파일에러가 발생한다.
  - 컴파일 시점에 값이 알려지지 않기 때문이다.
  - 컴파일 시점 상수를 요구하는 문맥에서 사용할 수 있어야 한다면 `const`가 아닌 `constexpr`를 사용해야한다.

## 함수에서의 constexpr
- `constexpr`을 함수 반환값에 사용할때는 두 가지 제약이 있다.
  - 가상으로 재정의된 함수가 아니어야 한다.
  - 반환값의 타입은 반드시 **리터럴 타입**이어야 한다.
- 함수에 `constexpr`을 붙일 경우 `inline`을 암시한다.
  - **컴파일 타임**에 평가되기 때문에 `inline` 함수들과 같이 컴파일 된다.
- 3<sup>n</sup>개를 담을 배열을 만들어야 하는 예제가 있다.
  - 컴파일 타임에 n 의 값을 알 수 있다면 **컴파일 단계**에서 배열의 사이즈를 잡는것이 유리할 것이다.
- `std::pow`의 경우 두가지 문제로 `std::array`의 크기를 지정하는 데에는 사용할 수 없다.
  - 부동소수점 형식에 대해 작동하지만 지금 필요한 것은 정수 결과이다.
  - `std::pow`는 `constexpr`이 아니다.
- 요구에 맞는 `pow` 함수를 직접 작성하면 해결된다.
- 구현은 제외하고 함수를 선언하고 사용하는 코드이다.
```cpp
constexpr
int pow(int base, int exp) noexcept
{
    ...
}

constexpr auto numConds = 5;
std::array<int, pow(3, numConds)> results; // results는 3^numConds개의 요소를 담는다.
```
- `pow` 앞에 `constexpr`이 있다고 해서 `pow`가 반드시 `const` 값을 돌려주는 것은 아니다.
- `base`와 `exp`가 컴파일 시점 상수일 때에만 `pow`의 결과를 컴파일 시점 상수로 사용할 수 있다는 뜻이다.

### C++11에서 구현한 pow
- `C++11`에서는 함수 본문에 지역 변수를 둘 수 없고 하나의 반환 표현식만이 와야 한다.
  - **삼항 연산자**와 **재귀**를 활용한다.
```cpp
constexpr int pow(int base, int exp) noexcept
{
    return (exp == 0 ? 1 : base * pow(base, exp - 1));
}
```

### C++14에서 구현한 pow
```cpp
constexpr int pow(int base, int exp) noexcept
{
    auto result = 1;
    for (int i = 0; i < exp; ++i) result *= base;
    return result;
}
``` 

## 멤버 함수에서의 constexpr
- `constexpr` 함수는 반드시 **리터럴 타입**을 받고 돌려주어야 한다.
- `C++11` 에서는 `void`를 제외한 모든 내장 형식이 리터럴 타입에 해당한다.

```cpp
class Point {
public:
    constexpr Point(double xVal = 0, double yVal = 0) noexcept
    : x(xVal), y(yVal)
    {}
    
    constexpr double xValue() const noexcept { return x; } // 조회용 멤버 함수(getter)
    constexpr double yValue() const noexcept { return y; } // 조회용 멤버 함수(getter)
    
    void setX(double newX) noexcept { x = newX; }
    void setY(double newY) noexcept { y = newY; }
    
private:
    double x, y;
};

constexpr Point p1(9.4, 27.7); // OK, constexpr 생성자가 컴파일 시점에서 실행됨
constexpr Point p2(28.8, 5.3;  // OK, constexpr 생성자가 컴파일 시점에서 실행됨

constexpr Point midpoint(const Point& p1, const Point& p2) noexcept
{
  return { (p1.xValue() + p2.xValue()) / 2, (p1.yValue() + p2.yValue()) / 2 };
}

constexpr auto mid = midpoint(p1, p2);
```
- 생성자를 `constexpr`로 선언할 수 있는 이유 ?
  - 인수들이 **컴파일 시점**에서 알려진다면 생성된 `Point`객체 멤버들의 값 역시 **컴파일 시점**에서 알려질 수 있기 때문이다.
- 컴파일 도중에 알려진 값으로 초기화된 `Point`객체에 대해 호출된다면 조회용 멤버 함수들 역시 `constexpr`이 될 수 있다.
- `Point`의 조회 함수들을 호출한 결과들로 `constexpr` 객체를 초기화하는 `constexpr`함수를 작성하는 것도 가능하다.

- `C++11`에서 `setX`와 `setY`를 `constexpr`로 선언할 수 없는 이유 ?
  - `constexpr`멤버 함수는 암묵적으로 `const`로 선언된다.
  - 멤버 함수들은 반환 형식이 `void`인데, `C++11`에서 `void`는 **리터럴 타입**이 아니다.

- `C++14`에서는 두 제약 모두 사라져 `setter`들도 `constexpr`이 될 수 있다.