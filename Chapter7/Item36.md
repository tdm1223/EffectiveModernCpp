# 항목 36. 비동기성이 필수일 때에는 std::launch::async를 지정하라
## 시동 방침(laynch policy)
- 일반적으로 `std::async`를 호출해서 어떤 함수(또는 호출 가능한 객체)를 실행 하는 것은 비동기적으로 실행하겠다는 의도가 깔려있다.
- `std::async`의 호출이 항상 그런 의미일 필요는 없다.
- `std::async` 호출은 함수를 어떤 **시동 방침**에 따라 실행한다는 좀 더 일반적인 의미를 가진다.
- 표준이 제공하는 시동 방침은 두 가지인데, `std::launch` 범위의 `enum`에 정의된 열거자들을 이용해서 지정할 수 있다.

### std::launch::async
- 함수는 반드시 **비동기적**으로(다른 스레드에서) 실행된다.

### std::launch::deferred
- 함수는 `std::async`가 돌려준 미래 객체(`std::future`)에 대해 `get`이나 `wait`가 **호출될 때에만 실행**될 수 있다.
- 함수는 호출이 일어날 때까지 지연된다.
  - `get`이나 `wait`가 호출되면 함수는 **동기적으로 실행**된다.
- 호출자는 함수의 실행이 종료될 때까지 차단된다.
  - `get`이나 `wait`가 호출되지 않으면 `std::async`에 전달된 함수는 절대로 실행되지 않는다.

### 기본 시동 방침
- 기본 시동 방침은 두가지 조건을 `OR`로 결합하여 사용한다.
```cpp
auto fut1 = std::async(f);
auto fut2 = std::async(std::launch::async | std::launch::deferred, f);
```
- 함수는 비동기적으로 실행될 수도 동기적으로 실행될 수도 있다. 
- 스레드 관리 구성요소들이 스레드의 생성과 파괴, 과다구독 회피, 부하 균형화의 책임을 떠맡을 수 있는 것은 이러한 유연성 덕분이다.
- `std::async`를 이용한 **동시적 프로그래밍을 편리하게 만들어주는 요인** 중 하나이다.

## std::async와 기본 시동 방침
- `std::async`를 기본 시동 방침과 함께 사용하면 몇가지 흥미로운 영향이 생긴다.
- 아래 문장이 스레드 `t`에서 실행된다고 가정한다.
```cpp
auto fut = std::async(f); // 기본 시동 방침으로 f를 실행한다
```
- `f`가 지연 실행될 수도 있으므로, `f`가 `t`와 **동시에 실행될지 예측하는 것이 불가능**하다.
- `f`가 `fut`에 대해 `get`이나 `wait`를 호출하는 스레드와는 다른 스레드에서 실행될지 예측하는 것이 불가능하다.
  - `t`가 그 스레드라고 할 때, `f`가 `t`와는 **다른 스레드에서 실행될지는 예측할 수 없다.**
- 프로그램의 모든 가능한 경로에서 `fut`에 대한 `get`이나 `wait`호출이 일어난다는 보장이 없을수도 있다.
  - `f`가 **반드시 실행될 것인지 예측하는 것이 불가능**할 수도 있다.
- `thread_local` 변수들의 사용과는 궁합이 잘 맞지 않는 경우도 많다.
  - `f`에 스레드 지역 저장소(`TLS`)를 읽거나 쓰는 코드가 있을때, 언제 접근하는지 예측이 불가능하기 때문이다.
```cpp
auto fut = std::async(f); // f의 TLS가 독립적인 스레드의 것일수도 있고 fut에 대해 get이나 wait를 호출하는 스레드의 것일 수도 있다.
```
- `wait` 기반 루프에도 영향을 미친다.
  - [지연된 과제](/Chapter7/Item35.md)에 대해 `wait_for` 나 `wait_until`을 호출하면 `std::future_status::deffered` 라는 값이 반환되기 때문이다.
  - 언젠가는 끝날 것처럼 보이는 루프가 **무한히 실행**될 수도 있다.
```cpp
using namespace std::literals;      // C++14의 시간 접미사들을 위해

void f()                            // f는 1초간 sleep 후 반환된다.
{
  std::this_thread::sleep_for(1s);
}

auto fut = std::async(f);           // f를 비동기적으로(개념상으로는) 실행한다.

while (fut.wait_for(100ms) !=       // f의 실행이 끝날 때까지 루프를 반복한다.
       std::future_status::ready)   // 그런데 실행이 끝나지 않을 수도 있다.
{
  ...
}
```
- `f`가 `std::async`를 호출한 스레드와 동시에 실행(f를 시동방침 std::launch::async로 실행했다면)되면 문제가 발생하지 않는다.
- `f`가 지연된다면 `fut.wait_for`는 항상 `std::future_status::deffered`를 반환한다.
  - 그 값은 `std::future_status::ready`와는 절대 같지 않으므로 **무한 루프가 발생**하게 된다.

### 해결 방법
- `std::async` 호출이 돌려준 미래 객체를 이용해서 해당 **과제가 지연되었는지 점검**하고 지연되었다면 시간 만료 기반 루프에 진입하지 않게 한다.
- **과제의 지연 여부**를 미래 객체로부터 직접 알아내는 방법은 없다.
  - `wait_for` 같은 **시간 만료 기반 함수**를 호출해야 한다.
  - 목적은 실제로 뭔가를 기다리는 것이 아니고 반환값이 `std::future_status::deferred`인지만 보면 된다.
- 만료 시간을 0으로 해서 `wait_for`를 호출하면 된다.
```cpp
auto fut = std::async(f);

if (fut.wait_for(0s) == std::future_status::deffered)  // 만일 과제가 지연되었으면...
{
  ...                                                  // fut에 wait나 get을 적용해서 f를 동기적으로 호출한다.
} else {
  while (fut.wait_for(100ms) !=                        // 무한 푸르는 불가능 하다
         std::future_status::ready) {                  // (f가 완료된다고 가정할 때)

    ...                                                // 과제가 지연되지도 않았고 준비되지도 않았으므로
                                                       // 준비될 때까지 동시적 작업을 수행한다.                     
  }
  
  ...                                                  // fut이 준비되었다.
}
```
  
## 기본 시동 방침과 std::async를 사용하는것이 적합한 경우
- 아래 조건들이 **모두 성립**할때 어떤 과제에 대해 기본 시동 방침과 함께 `std::async`를 사용하는 것이 적합하다.
  - 과제가 `get` 이나 `wait` 를 호출하는 스레드와 반드시 동시적으로 실행되어야 하는 것은 아니다.
  - 여러 스레드 중 어떤 스레드의 `thread_local` 변수들을 읽고 쓰는지가 중요하지 않다.
  - `std::async`가 돌려준 미래 객체에 대해 `get`이나 `wait`가 반드시 호출된다는 보장이 있거나, 과제가 전혀 실행되지 않아도 괜찮다.
  - 과제가 지연된 상태일 수도 있다는 점이 `wait_for`나 `wait_until`을 사용하는 코드에 반영되어 있다.
- 위 조건 중에서 하나라도 성립하지 않는다면 `std::async`가 주어진 과제를 **반드시 비동기로 수행**하도록 변경해야 한다.

## 시동 방침을 활용해 비동기로 수행하기
- `std::launch::async`를 첫 인수로 지정해서 `std::async`를 호출하면 된다.
- 사용할때마다 시동 방침을 명시적으로 지정하지 않기 위해 아래 템플릿 코드를 활용하면 된다.
```cpp
// C++11
template<typename F, typename... Ts>
inline std::future<typename std::result_of<F(Ts...)>::type> reallyAsync(F&& f, Ts&&... params)
{
  return std::async(std::launch::async,
                    std::forward<F>(f),
                    std::forward<Ts>(params)...);
}

// C++14
template<typename F, typename... Ts> inline auto reallyAsync(F&& f, Ts&&... params)
{
  return std::async(std::launch::async,
                    std::forward<F>(f),
                    std::forward<Ts>(params)...);
}

auto fut = reallyAsync(f);
```
- [호출 가능 객체 f와 0개 이상의 매개변수들로 이루어진 매개변수 묶음 params를 받아서 std::async에 완벽하게 전달한다.](/Chapter5/Item25.md)
  - 이때 `std::launch::async`를 시동 방침으로 지정한다.
- `params`로 `f`를 호출한 결과를 나타내는 `std::future` 객체를 돌려준다.
  - 결과의 형식을 결정하는 것은 쉽다.
  - 형식 특질 `std::result_of`를 적용한 결과가 그 형식이다.
- `C++14`에서는 `reallyAsync`의 반환 형식을 `auto`로 연역할 수 있어서 좀 더 간단하다.