# 항목 12. 재정의 함수들을 override로 선언하라
## 오버라이딩의 필수 조건
- 기본 클래스 함수가 반드시 가상 함수여야 한다.
- 기본 클래스 함수와 파생 클래스 함수의 이름이 반드시 동일해야 한다.
- 기본 클래스 함수와 파생 클래스 함수의 매개변수 형식들이 반드시 동일해야 한다.
- 기본 클래스 함수와 파생 클래스 함수의 `const`성이 반드시 동일해야 한다.
- 기본 클래스 함수와 파생 클래스 함수의 반환 형식과 예외 명세가 반드시 호환되어야 한다.
- 멤버 함수들의 참조 한정사들이 반드시 동일해야 한다.(`C++11`에서 추가된 조건)
  - 참조 한정사를 사용하면 멤버 함수를 왼값에만 또는 오른값에만 사용할 수 있게 제한할 수 있다.
  - 가상 함수가 아닌 멤버 함수에도 멤버 함수 참조 한정사를 적용할 수 있다.

```cpp
class Base 
{
public:
    virtual void f1() const;
    virtual void f2(int x);
};

class Derived : public Base 
{
public:
    virtual void f1();
    virtual void f2(unsigned int x);
};
```
- `f1`에서는 `const`여부, `f2`에서는 매개변수 종류에 의해 두 함수는 오버라이딩 되지 않지만 컴파일러는 위 코드에서 경로를 출력해 주지 않는다.

## override
- 위 처럼 파생 클래스 재정의 선언은 프로그래머가 실수하기 쉽다.
- `C++11`은 파생 클래스 함수가 기본 클래스의 버전을 재정의 하려 한다는 의도를 병시적으로 표현하는 방법을 제공한다.
- 파생 클래스 함수를 `override`로 선언 하면 된다.
```cpp
class Base
{
public:
    virtual void f1() const;
    virtual void f2(int x);
};

class Derived: public Base
{
public:
    virtual void f1() override;                  // const를 빼먹음
    virtual void f2(unsigned int x) override;    // int -> unsigned int x
};
```
- 재정의 인식을 하지 못하기 때문에 위 코드는 컴파일 되지 않는다.
- `override`라는 식별자는 멤버 함수 선언의 끝에 나올 때에만 예약된 의미를 가진다.
  - `override`라는 이름을 사용하는 코드가 남아 있어도 `C++11` 적용을 위해 이름을 변경할 필요는 없다.
