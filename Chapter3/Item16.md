# 항목 16. const 멤버 함수를 스레드에 안전하게 작성하라
## const 멤버 함수
- 멤버 함수가 멤버 변수들을 수정하지 않는다면 `const`로 선언하는 것이 맞다.
- 여러 스레드가 하나의 객체에 대해 `const` 멤버 함수를 동시에 실행하려 한다면 문제가 생길 수 있다.

### 문제가 발생할 수 있는 코드
```cpp
class Polynomial {
public:
    using RootsType = std::vector<double>; // 근을 담는 자료구존
    
    RootsType roots() const
    {
        if(!rootsAreValid)          // 캐시가 유효하지 않다면 
        {
            ...                     // 근들을 계산해서 rootVals에 저장한다.
            rootsAreValid = true;
        }
        return rootVals;
    }

private:
    mutable bool rootsAreValid{ false };
    mutable RootsType rootVals{};
};
```
- `roots`는 개념적으로 자신이 속한 `Polynomial` 객체를 변경하지 않는다.
  - 캐싱을 위해서는 `rootVals`과 `rootAreValid`의 변경이 필요할 수 있다.
  - 두 변수를 **mutable**로 선언하는 이유이다.
- 두 스레드가 하나의 `Polynomial`객체에 대해 `roots`를 **동시에 호출**하면 문제가 발생할 수 있다.
  - `roots`안에서 두 스레드 중 하나 또는 둘 모두 멤버 `rootsAreValid`와 `rootVals`를 **수정** 할 수 있기 때문이다.
- `roots`가 `const`로 선언되어 있지만 스레드에 안전하지 않다.
  - 스레드 안전성의 결여를 수정해야 한다.

### 해결책 : mutex의 사용
```cpp
class Polynomial {
public:
    using RootsType = std::vector<double>;
    
    RootsType roots() const
    {
        std::lock_guard<std::mutex> g(m);     // 뮤텍스를 잠근다.
        if(!rootsAreValid)
        {
            ...
            rootsAreValid = true;
        }
        return rootVals;
    }                                         // 뮤텍스를 푼다.
    
private:
    mutable std::mutex m;
    mutable bool rootsAreValid{ false };
    mutable RootsType rootVals{};
};
```
- `std::mutex` 형식의 객체 `m`은 `mutable`로 선언되었다.
  - `m`을 잠그고 푸는 멤버 함수들은 `비const`이지만 `roots` 안에서는 `m`이 `const`객체로 간주되기 때문이다.
- `mutex`를 추가함으로써 `Polynomial`객체는 **복사와 이동이 불가능**해 졌다.

## atomic 카운터
- 뮤텍스를 도입하는 것이 너무 과할 수도 있다.
- 함수의 호출 횟수를 세고 싶다면 [atomic 카운터](/Chapter7/Item40.md)를 사용해서 비용을 줄일 수 있는 경우가 많다.
- `std::atomic`도 **복사와 이동이 불가능**하다.

```cpp
class Point {
public:
    double distanceFromOrigin() const noexcept
    {
        ++callCount;
        return std::hypot(x, y);
    }
private:
    mutable std::atomic<unsigned> callCount { 0 };
    double x, y;
};
```
- `Point`에 `callCount`를 도입하면 `Point` 역시 복사와 이동이 불가능해 진다.

## std::atomic의 문제
- `std::atomic` 변수에 대한 연산들이 뮤텍스를 획득하고 해제하는 것보다 비용이 싸다.
  - `std::atomic` 변수를 남용할 가능성이 있다.
- 계산 비용이 큰 값을 캐시에 저장하는 클래스라면 **뮤텍스** 대신 **한 쌍의 std::atomic 변수**를 사용할만 하다 생각할 수 있다.
```cpp
class Widget {
public:
    ...
    int magicValue() const
    {
        if (cacheValid)
        {
            return cachedValue;
        }
        else 
        {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cachedValue = val1 + val2;
            cacheValid = true; // true로 바꿨지만 다른 스레드는 false로 보고 계산을 진행한다.
            return cachedValue;
        }
    }
private:
    mutable std::atomic<bool> cacheValid{ false };
    mutable std::atomic<int> cachedValue;
};
```
- 여러개의 스레드가 동시에 `magicValue()` 에 접근하게 되면 비싼 계산을 하는 함수를 여러 스레드가 사용하게 된다.
  - 한 스레드가 `magicValue`를 호출 후 `cacheValid`가 `false`이므로 비용이 큰 두 계산 수행 후 둘의 합을 `cachedValue`에 저장한다.
  - 다른 스레드가 `magicValue`를 호출하는데 `cacheValid`가 `false`이므로 첫 번째 스레드가 한 계산을 수행한다.
- `cachedValue`와 `cacheValid`의 배정 순서를 반대로 바꿔도 문제는 동일하다.
- **둘 이상의 변수나 메모리 장소**를 하나의 단위로서 조작해야 할 때에는 뮤텍스를 사용하는것이 적합하다.
- 동기화가 필요한 **변수가 하나이거나 메모리 장소 하나**에 대해서는 `std::atomic`을 사용하는 것이 적합하다.