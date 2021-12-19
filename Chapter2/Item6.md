# 항목 6. auto가 원치 않은 형식으로 연역될 때에는 명시적 형식의 초기치를 사용하라
## auto 형식 연역의 예외
```cpp
std::vector<bool> features(const Widget& w){} // Widget을 하나 받고 std::vector<bool>을 돌려주는 함수

Widget w;
bool highPriority = features(w)[5];
auto highPriority = features(w)[5];

processWidget(w, highPriority); // w를 우선순위에 맞게 처리한다.

```
- 위 코드는 잘작동한다. `highPriority`의 명시적 형식을 `auto`로 대체하면 어떨까?
  - `auto`를 사용하면 `highPriority`의 형식은 더 이상 `bool`이 아니게 된다!
  - `std::vector<bool>`의 `operator[]`가 돌려주는 것은 그 컨테이너의 한 요소에 대한 참조가 아니라 `std::vector<bool>::reference` 형식의 객체이다.


### std::vector<bool>::reference
- `std::vector<bool>::reference`는 **대리자 클래스**로 다른 어떤 형식의 행동을 흉내 내고 보강하는 것이 존재 이유인 클래스의 예이다.
- `std::vector<bool>::reference`가 존재하는 것은 `std::vector<bool>`이 자신의 항목들을 `bool`당 1비트의 압축된 형태로 표현하도록 명시되어 있기 때문이다.
  - `std::vector<bool>`의 `operator[]`를 직접적으로 구현할 수 없다.
- `std::vector<T>의 operator[]`는 `T&`를 돌려주도록 되어있지만 `C++`에서 비트에 대한 참조는 금지되어 있다.
  - `bool&`을 직접 돌려줄 수 없기 때문에 `bool&`처럼 작동하는 객체를 돌려주는 우회책을 사용한다.
  - 우회책이 통하려면 `bool&`가 쓰이는 모든 문맥에서 `std::vector<bool>::reference` 객체를 `bool&` 처럼 사용할 수 있어야 한다.
    - 이를 가능하게 하는 `std::vector<bool>::reference`의 기능 중 하나가 `bool`로의 암묵적 변환이다.

### bool highPriority = features(w)[5];
- `features`는 `std::vector<bool>` 객체를 돌려주며 객체에 대해 `operator[]`가 호출된다.
- `operator[]`는 `std::vector<bool>::reference` 객체를 돌려주며 그 객체가 암묵적으로 `bool`로 변환되어 `highPriority`의 초기화에 쓰인다.
- 결과적으로 `highPriority`는 `features`가 돌려준 `std::vector<bool>`의 5번 비트의 값을 가지게 된다.

### auto highPriority = features(w)[5]; 
- `features`는 `std::vector<bool>`객체를 돌려주며 그 객체에 대해 `operator[]`가 호출된다.
- `operator[]`가 `std::vector<bool>::reference` 객체를 돌려준다.
- `auto`에 의해 `highPriority`의 형식이 연역되기 때문에, `highPriority`가 `features`가 돌려준 `std::vector<bool>`의 5번 비트로 초기화 되지 않는다.

### highPriority의 초기화
- `features` 호출은 임시 `std::vector<bool>` 객체를 돌려준다.
- 이 임시 객체에 대해 호출된 `operator[]`는 ``std::vector<bool>::reference`` 객체를 돌려준다.
  - 객체에는 임시객체가 관리하는 비트들을 담은 자료구조의 한 **워드를 가리키는 포인터**와 그 워드에서 참조된 비트에 해당하는 **비트의 오프셋**이 담겨 있다.
- `highPriority`는 `std::vector<bool>::reference` 객체의 한 복사본이며 따라서 `highPriority`역시 임시객체 안의 해당 워드를 가리키는 포인터와 비트의 오프셋을 담게 된다.
  - 하지만 위에서 생성된 임시객체는 문장의 끝에서 파괴되고 `highPriority`의 포인터는 대상을 잃은 포인터가 되게 된다.
    - 결과적으로 `processWidget`호출은 미정의 행동을 유발하게 된다.
- `auto`는 **보이지 않는 대리자 클래스 형식의 표현식**을 대입하는 것을 피해야 한다.

## static_cast를 통한 해결
- 문제를 해결하기 위해 사용하는 것이 타입 캐스팅이다.
  - `static_cast`를 사용하면 위와 같은 문제를 방지할 수 있다.

```cpp
auto highPriority = static_cast<bool>(features(w)[5]);
```

- `highPriority`는 `bool` 타입이 되어 위와 같은 문제를 회피 할 수 있다.
