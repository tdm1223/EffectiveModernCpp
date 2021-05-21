# EffectiveModernCpp

# 형식 연역
- [항목1. 템플릿 형식 연역 규칙을 숙지하라](/Chapter1/Item1.md)
- [항목2. auto의 형식 연역 규칙을 숙지하라](/Chapter1/Item2.md)
- [항목3. decltype의 작동 방식을 숙지하라](/Chapter1/Item3.md)
- [항목4. 연역된 형식을 파악하는 방법을 알아두라](/Chapter1/Item4.md)

# auto
- [항목5. 명시적 형식 선언보다는 auto를 선호하라](/Chapter2/Item5.md)
- [항목6. auto가 원치 않은 형식으로 연역될 때에는 명시적 형식의 초기치를 사용하라](/Chapter2/Item6.md)

# 현대적 C++에 적응하기
- [항목7. 객체 생성 시 괄호와 중괄호를 구분하라](/Chapter3/Item7.md)
- [항목8. 0과 NULL보다 nullptr를 선호하라](/Chapter3/Item8.md)
- [항목9. typedef보다 별칭 선언을 선호하라](/Chapter3/Item9.md)
- [항목10. 범위 없는 enum보다 범위 있는 enum을 선호하라](/Chapter3/Item10.md)
- [항목11. 정의되지 않은 비공개 함수보다 삭제된 함수를 선호하라](/Chapter3/Item11.md)
- [항목12. 재정의 함수들을 override로 선언하라](/Chapter3/Item12.md)
- [항목13. iterator보다 const_iterator를 선호하라](/Chapter3/Item13.md)
- [항목14. 예외를 방출하지 않을 함수는 noexcept로 선언하라](/Chapter3/Item14.md)
- [항목15. 가능하면 항상 constexpr을 사용하라](/Chapter3/Item15.md)
- [항목16. const 멤버 함수를 스레드에 안전하게 작성하라](/Chapter3/Item16.md)
- [항목17. 특수 멤버 함수들의 자동 작성 조건을 숙지하라](/Chapter3/Item17.md)

# 똑똑한 포인터
- [항목18. 소유권 독점 자원의 관리에는 std::unique_ptr를 사용하라](/Chapter4/Item18.md)
- [항목19. 소유권 공유 자원의 관리에는 std::shared_ptr를 사용하라](/Chapter4/Item19.md)
- [항목20. std::shared_ptr처럼 작동하되 대상을 잃을 수도 있는 포인터가 필요하면 std::weak_ptr를 사용하라](/Chapter4/Item20.md)
- [항목21. new를 직접 사용하는 것보다 std::make_unique와 std::make_shared를 선호하라](/Chapter4/Item21.md)
- [항목22. Pimpl 관용구를 사용할 때에는 특수 멤버 함수들을 구현 파일에서 정의하라](/Chapter4/Item22.md)

# 오른값 참조, 이동 의미론, 완벽 전달
- [항목23. std::move와 std::forward를 숙지하라](/Chapter5/Item23.md)
- [항목24. 보편 참조와 오른값 참조를 구별하라](/Chapter5/Item24.md)
- [항목25. 오른값 참조에는 std::move를, 보편 참조에는 std::forward를 사용하라](/Chapter5/Item25.md)
- [항목26. 보편 참조에 대한 중복적재를 피하라](/Chapter5/Item26.md)
- [항목27. 보편 참조에 대한 중복적재 대신 사용할 수 있는 기법들을 알아 두라](/Chapter5/Item27.md)
- [항목28. 참조 축약을 숙지하라](/Chapter5/Item28.md)
- [항목29. 이동 연산이 존재하지 않고, 저렴하지 않고, 적용되지 않는다고 가정하라](/Chapter5/Item29.md)
- [항목30. 완벽 전달이 실패하는 경우들을 잘 알아두라](/Chapter5/Item30.md)

# 람다 표현식
- [항목31. 기본 갈무리 모드를 피하라](/Chapter6/Item31.md)
- [항목32. 객체를 클로저 안으로 이동하려면 초기화 갈무리를 사용하라](/Chapter6/Item32.md)
- [항목33. std::forward를 통해서 전달할 auto&& 매개변수에는 decltype을 사용하라](/Chapter6/Item33.md)
- [항목34. std::bind보다 람다를 선호하라](/Chapter6/Item34.md)

# 동시성 API
- [항목35. 스레드 기반 프로그래밍보다 과제 기반 프로그래밍을 선호하라](/Chapter7/Item35.md)
- [항목36. 비동기성이 필수일 때에는 std::launch::async를 지정하라](/Chapter7/Item36.md)
- [항목37. std::thread들을 모든 경로에서 합류 불가능하게 만들어라](/Chapter7/Item37.md)
- [항목38. 스레드 핸들 소멸자들의 다양한 행동 방식을 주의하라](/Chapter7/Item38.md)
- [항목39. 단발성 사건 통신에는 void 미래 객체를 고려하라](/Chapter7/Item39.md)