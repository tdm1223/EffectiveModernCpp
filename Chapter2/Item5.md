# 항목 5. 명시적 형식 선언보다는 auto를 선호하라
## auto 사용의 이점
### 초기화를 빼먹는 실수
- 변수를 선언만하고 초기화를 하지 않는 경우가 생길 수 있다.
- 이런 실수 때문에 예상치 못한 문제들이 발생할 수 있어 항상 조심해야한다.
- `auto`를 사용한다면 **컴파일러가 에러를 발생**시켜 실수가 발생할 여지를 남기지 않는다.

```cpp
int x1;       // 초기화 되지 않을 수 있다.
auto x2;      // 컴파일 에러
auto x3 = 0;  // 양호함: x3의 값이 잘 정의됨
```

### 컴파일러만 알던 형식 지정 가능
- 람다 표현식으로 선언된 변수의 경우에는 우리가 형식을 지정할 수 없다. 
  - 컴파일러만 알고 있는 형식이기 때문이다.
- [auto는 형식 연역을 사용하므로](/Chapter1/Item2.md) 컴파일러만 알던 형식을 지정할 수 있다.

```cpp
auto derefUPLESS =                        // std::unique_ptr들이
    [](const std::unique_ptr<Widget>& p1, // 가리키는 Widget
       const std::unique_ptr<Widget>& p2) // 객체들을 비교하는
    { return *p1 < *p2; };                // 함수
```
- 클로저를 담는 변수를 선언할 때 굳이 auto를 사용할 필요는 없다는 생각이 들수도 있다.
- `std::function` 객체를 사용하면 되지 않을까?
```cpp
std::function<bool(const std::unique_ptr<Widget>&, const std::unique_ptr<Widget>&)>
    auto derefUPLESS = [](const std::unique_ptr<Widget>& p1, const std::unique_ptr<Widget>& p2) { return *p1 < *p2; };
```
- `std::function`으로 선언된 변수의 형식은 `std::function` 템플릿의 한 인스턴스이며 크기는 임의의 주어진 서명에 대해 고정되어 있다.
- 크기가 요구된 클로저를 저장하기에 부족할 수 있고 이때 `std::function`은 힙 메모리를 할당해서 클로저를 저장한다.

### 형식 단축의 문제 회피
```cpp
std::vector<int> v;
unsigned sz = v.size(); // 운영체제에 따라 문제 발생 가능
auto sz = v.size():   	// auto를 사용하여 해결
```
- `std::vector` 의 반환 형식은 `std::vector<int>::size_type` 이다.
- 만약 `unsigned` 타입을 사용했다면 운영체제에 따라 **32비트**가 될수도 **64비트**가 될수도 있는 문제가 있다.

```cpp
std::unordered_map<std::string, int> m;
for (const std::pair<std::string, int>& p : m)
{
    ...
}

// auto를 사용
for (const auto& p : m)
{
    ...
}
```
- `std::unordered_map`에 담겨있는 형식은 `pair<const std::string, int>`이므로 의도치 않은 형변환이 일어난다.
- `auto`를 사용한다면 더 효율적이고 타자도 간편하며 실수할 여지가 줄어든다.

### auto는 완벽한 것인가?
- `auto` 변수의 형식은 변수의 초기화에 쓰이는 표현식으로부터 연역되는데, 초기화 표현식의 형식이 기대하지 않거나 바람직하지 않은 것일 경우도 있다.
- `auto`를 사용하면 가독성의 문제도 존재한다.
  - 예전에는 소스코드를 조금만 봐도 객체의 형식을 파악할 수 있었다면 `auto`사용시 객체의 형식 파악이 힘들어 질 수 있다.
  - 이 문제는 [IDE의 기능](/Chapter1/Item4.md)으로 완화되는 경우가 많다.
