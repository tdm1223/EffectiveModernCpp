# 항목 8. 0과 NULL보다 nullptr를 선호하라
- `C++`에서 리터럴 0은 `int`이지 포인터가 아니다.
  - `C++`의 기본 방침은 0은 int이지 포인터가 아니라는 것이다.
- 0과 마찬가지로 `NULL`도 포인터 형식이 아니다.
  - 구현(컴파일러)이 `NULL`에 `int` 이외의 정수 형식을 부여할 수 있다.

## nullptr
- `nullptr`의 형식으로 정의된다.
  - 정수 형식이 아니다.
  - 포인터 형식도 아니다.
- `nullptr`는 모든 `raw` 포인터 형식으로 암묵적으로 변환되며, `nullptr`는 모든 형식의 포인터처럼 행동한다.

### 포인터를 받는 버전의 함수 오버로딩
- 0과 `NULL` 대신 `nullptr`를 사용하면 오버로딩이 예상과 다르게 해소되는 일이 없다.
```cpp
void f(int);
void f(bool);
void f(void*);

f(0); // f(int) 호출
f(NULL); // 컴파일 되지 않을 수도 있지만 보통은 f(int) 호출
f(nullptr); // f(void*) 호출
```
- `nullptr`가 절대로 정수 형식으로 해석되지 않기 때문에 `f(void*)`가 호출된다.

### 코드의 명확성 향상
- `auto`가 관여하는 상황에서 코드의 명확성을 특히 높여준다.
```cpp
auto result = findRecord(/*인수*/);

// 정수랑 비교
if(result == 0)
{
    ...
}

// nullptr랑 비교
if(result == nullptr)
{
    ...
}
```
- `findRecord`의 형식을 모른다면 `result`가 포인터 형식인지 정수 형식인지를 명확히 말할 수 없게 된다.
  - `result == 0`에서 0은 포인터 형식으로도, 정수 형식으로도 작용할 수 있다.
- `nullptr`를 사용하면 중의성을 없애준다.

### 템플릿에서의 nullptr
- 적절한 뮤텍스를 잠근 상태에서만 호출해야 하는 함수들이 있는데 그 함수들이 각자 다른 종류의 포인터를 받는 예가 있다.
```cpp
int f1(std::shared_ptr<Widget> spw);
double f2(std::unique_ptr<Widget> upw);
bool f3(Widget* pw);

std::mutex f1m, f2m, f3m; // 뮤텍스들
using MuxGuard = std::lock_guard<std::mutex>;

{
    MuxGuard g(f1m);            // f1용 뮤텍스를 잠근다.
    auto result = f1(0);        // 0을 널 포인터로서 f1에 전달
}                               // 뮤텍스가 풀리는 위치

{
    MuxGuard g(f2m);            // f2용 뮤텍스를 잠근다.
    auto result = f2(NULL);     // NULL을 널포인터로서 f2에 전달
}                               // 뮤텍스가 풀리는 위치

{
    MuxGuard g(f3m);            // f3용 뮤텍스를 잠근다.
    auto result = f3(nullptr);  // nullptr를 널포인터로서 f3에 전달
}                               // 뮤텍스가 풀리는 위치
```
- 위에서 중복 되는 코드를 템플릿화 해보자
  
```cpp
template<typename FuncType, typename MuxType, typename PtrType>
decltype(auto) lockAndCall(FuncType func, MuxType& mutex, PtrType ptr)
{
    using MuxGuard = std::lock_guard<MuxType>;
    MuxGuard g(mutex);
    return func(ptr);
}

auto result = lockAndCall(f1, f1m, 0);        // 오류
auto result = lockAndCall(f2, f2m, NULL);     // 오류
auto result = lockAndCall(f3, f3m, nullptr);  // 성공
```
- 반환 형식(`decltype(auto)`)이 이해가 안된다면 [항목3](/Chapter1/Item3.md)을 다시 보자
- 첫 번째 호출에서 0을 넘겨주면 컴파일러가 템플릿 형식 연역을 적용해 0을 `int`로 적용한다.
- 두 번째 호출도 `NULL`을 넘겨주면 매개변수 `ptr`는 정수 형식으로 연역되어 `std::unique_ptr<Widget>`을 기대하는 `f2`에 `ptr`를 넘겨주어 형식 오류가 발생한다.
- 세 번째 호출은 `nullptr`을 넘겨주기 `std::nullptr_t에서 Widget*`로의 암묵적 변환이 일어나 문제가 없다.
