# 항목 37. std::thread들을 모든 경로에서 합류 불가능하게 만들어라
## std::thread 객체의 상태
- 모든 `std::thread` 객체는 **합류 가능**(`joinable`) 상태이거나 **합류 불가능**(`unjoinable`) 상태이다.

### 합류 가능 std::thread
- 바탕 실행 스레드 중 현재 **실행 중**이거나 **실행 중 상태로 전이**할 수 있는 스레드에 대응된다.
- 차단된 상태이거나 실행 일정을 기다리는 중인 바탕 스레드에 해당하는 `std::thread`는 합류 가능 상태이다.
- 실행이 완료된 바탕 스레드에 해당하는 `std::thread`객체도 합류 가능으로 간주한다.

### 합류 불가능 std::thread
- 합류할 수 없는 `std::thread`를 말한다. 합류 불가능한 객체로는 아래와 같은 것들이 있다.
- **기본 생성**된 `std::thread`
  - 실행할 함수가 없으므로 바탕 실행 스레드와는 대응되지 않는다.
- 다른 `std::thread` 객체로 **이동된 후**의 `std::thread` 객체
  - 이동의 결과로 원본 `std::thread`에 대응되던 바탕 스레드는 대상 `std::thread`의 바탕스레드가 된다.
- `join`에 의해 합류된 `std::thread`
  - `join` 이후의 `std::thread` 객체는 실행이 완료된 바탕 실행 스레드에 대응되지 않는다.
- `detach`에 의해 탈착된 `std::thread`
  - `detach`는 `std::thread` 객체와 그에 대응되는 바탕 스레드 사이의 연결을 끊는다.

## std::thread의 합류 가능성이 중요한 이유
- 합류 가능성이 중요한 이유는 **합류 가능 스레드의 소멸자가 호출되면 프로그램이 종료**되기 때문이다.
- 필터링 함수 `filter`와 최댓값 `maxVal`을 매개변수들로 받는 함수 `doWork`가 있다고 가정한다.
  - `doWork`는 계산에 필요한 조건이 만족되는지 점검한 후 0 이상 `maxVal` 미만의 모든 값 중 필터를 통과한 값들에 대해 계산을 수행한다.
  - 필터링과 조건 점검이 오래 걸린다면 두 작업을 동시에 실행하는 것이 합리적이다.
- 기본적으로는 과제 기반 설계를 적용하는 것이 좋겠지만 필터링을 수행하는 스레드의 우선순위를 설정해야 할 필요가 있다고 가정한다.
  - 스레드 우선순위를 설정하려면 스레드의 네이티브 핸들이 필요하다.
  - 네이티브 핸들은 오직 `std::thread API`를 통해서 얻을 수 있다.
```cpp
constexpr auto tenMillion = 10'000'000;

bool doWork(std::function<bool(int)> filter, int maxVal = tenMillion) // 계산 수행 여부를 돌려준다.
{
  std::vector<int> goodVals; // 필터를 통과한 값들
   
  std::thread t([&filter, maxVal, &goodVals] {                 // goodVals에 값들을 채운다.
                  for (auto i = 0; i <= maxVal; ++i)
                  { if (filter(i)) goodVals.push_back(i); }
                });

  auto nh = t.native_handle();
  ...
  
  if (conditionsAreSatisfied()) {                               // 조건들이 만족되었다면
    t.join();                                                   // t의 완료를 기다린다.
    performComputation(goodVals);
    return true;                                                // 계산이 수행되었음을 뜻하는 값 반환
  }
  
  return false;
}
```
- `conditionsAreSatisfied()`가 `true`를 반환하면 문제는 발생하지 않는다.
- `false`를 반환하거나 예외를 던진다면 실행의 흐름이 `doWork`의 끝에 도달한다.
  - `t`의 **소멸자가 호출**되고 이때 `t`가 여전히 **합류 가능 상태**이기 때문에 프로그램이 종료된다.
- `std::thread`의 소멸자가 왜 이런식으로 행동하는지 궁금할수도 있다.
  - 이유는 다른 두 옵션은 명백히 더 나쁘기 때문이다.

### 다른 옵션 1 : 암묵적 join
- `std::thread`의 소멸자가 바탕 비동기 실행 스레드의 완료를 기다리게 하는 것이다.
- 합리적으로 보이지만 실제로는 **추적하기 어려운 성능 이상들**이 나타날 수 있다.
- `conditionsAreSatisfied()` 가 이미 `false`를 반환했는데도 모든 값에 필터가 적용되길 `doWork`가 기다리는 것은 직관적이지 않다.

### 다른 옵션 2 : 암묵적 detach
- `std::thread`의 소멸자가 **std::thread 객체**와 **바탕 실행 스레드** 사이의 연결을 끊게 하는 것이다.
- 바탕 스레드가 실행을 계속 할 수 있다.
- 합리적으로 보이지만 `join`의 경우보다 더 나쁜 디버깅 문제를 일으킨다.
- `conditionsAreSatisfied()`가 `false`를 반환하여 `doWork`함수가 **종료되었다고 가정**한다.
- `doWork`가 종료되면 지역변수들도 함께 소멸된다.
- 스레드는 `goodVals`라는 지역변수를 참조로 사용하고 있다.
  - 스레드는 소멸된 변수에 접근하게 되어 미정의 행동을 하게 되는데, 이를 디버깅하기는 매우 어렵다.

### 결론과 해결책
- 합류 가능 스레드를 파괴했을 떄의 결과가 매우 절망적이므로 표준 위원회에서는 **파괴를 금지**하기로 했다.
  - 합류가능 스레드를 파괴하면 프로그램이 종료된다.
- `RAII`방식을 사용하여 문제를 해결할 수 있다.
  - `std::thread` 객체에 대한 표준 `RAII`는 없기 때문에 작성해야 한다.
```cpp
class ThreadRAII {
public:
  enum class DtorAction { join, detach };
  
  ThreadRAII(std::thread&& t, DtorAction a)          // 소멸자에서 t에
  : action(a), t(std::move(t)) {}                    // 대해 동작 a를 수행
  
  ~ThreadRAII()
  {
    if (t.joinable()) {
      if (action == DtorAction::join) {
        t.join();
      } else {
        t.detach();
      }
    }
  }
  
  ThreadRAII(ThreadRAII&&) = default;
  ThreadRAII& operator=(ThreadRAII&&) = default;
  
  std::Thread& get() { return t; }
  
private:
  DtorAction action;
  std::thread t;
};
```
- 생성자는 `std::thread` **오른값**만 받는다.
  - `ThreadRAII` 객체로 이동할 것이기 때문이다.
- 생성자의 매개변수들은 호출자가 직관적으로 기억할 수 있는 순서로 선언되어 있다.
  - 멤버 초기화 목록은 자료 멤버들이 선언된 순서를 따른다.
- `ThreadRAII`는 바탕 `std::thread` 객체에 접근할 수 있는 `get` 함수를 제공한다.
- `ThreadRAII` 소멸자는 `std::thread` 객체 `t`에 대해 멤버 함수를 호출하기 전에 `t`가 합류 가능인지부터 점검한다.
  - 합류 불가능 스레드에 대해 `join`이나 `detach`를 호출하면 미정의 행동이 나오기 때문에 꼭 필요하다.
- [ThreadRAII는 소멸자를 선언하므로 컴파일러가 이동 연산들을 작성해주지 않는다.](/Chapter3/Item17.md)
  - `ThreadRAII` 객체의 이동을 지원하지 않을 이유는 없기에 명시적으로 요청하였다.
- `t.joinable()`의 실행과 `join` 또는 `detach` 호출 사이에 **다른 스레드가 t를 합류 불가능하게 만들면** 경쟁 조건이 성립하지 않을까라고 생각할수 있다.
  - 합류 가능한 `std::thread`객체는 오직 **멤버 함수 호출** 또는 **이동 연산**에 의해서만 합류 불가능한 상태로 변할 수 있다.

### doWork에 ThreadRAII를 적용한 코드
```cpp
bool doWork(std::function<bool(int)> filter, int maxVal = tenMillion)
{
  std::vector<int> goodVals;
   
  ThreadRAII t(
    std::thread([&filter, maxVal, &goodvals]
                {
                  for (auto i = 0; i <= maxVal; ++i)
                    { if (filter(i)) goodVals.push_back(i); }
                }),
                ThreadRAII::DtorAction::join // RAII 동작
  );

  auto nh = t.get().native_handle();
  ...
  
  if (conditionsAreSatisfied()) {
    t.get().join();
    performComputation(goodVals);
    return true;
  }
  
  return false;
}
```
- `ThreadRAII` 소멸자가 비동기 실행 스레드에 대해 `join`을 호출하도록 했는데 `detach`를 호출하면 디버깅 문제가 발생할 수 있다.
- `join`이 **성능 이상**을 유발할수 있지만 **미정의 행동**, **프로그램 종료**, **성능 이상**중 그나마 성능 이상이 제일 덜 나쁘다.
  - `join`이 실행되게 하면 성능 이상만이 아니라 [프로그램이 멈추는 문제](/Chapter7/Item39.md)까지 발생할 수도 있다.
  - 문제에 대한 제대로된 해결책은 **비동기적으로 실행되는 람다에게 일찍 반환하라고 알려주는 것**이다.