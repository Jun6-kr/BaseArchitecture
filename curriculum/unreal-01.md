# Unreal 01 학습 기록 - 리플렉션, 인터페이스, 가비지 컬렉션

- 테스트 진행일: 2026-10-07
- 오답 분석 완료일: 2026-10-13
- 진행 방식: 테스트 선택 이유 확인 → 개념 분리 → 교정 답변
- 결과: **리플렉션 용어와 인터페이스 구조 보완(🟡), 가비지 컬렉션 개념 통과(🟢)**
- 다음 복습: 2026-10-14, 2026-10-20, 2026-11-12

## 테스트에서 확인한 범위

- `TEXT`, PascalCase, `int32`, `FString`
- UHT와 `generated.h`
- 리플렉션과 `UCLASS`, `USTRUCT`, `UENUM`
- 언리얼 인터페이스의 `U` 클래스와 `I` 클래스
- CDO와 `CreateDefaultSubobject`
- 멀티캐스트 델리게이트
- `TArray`, `TSet`과 복잡도
- `UObject` 가비지 컬렉션
- JSON, 에셋, 프로젝트 폴더

## 오답 분석 1 - 리플렉션

### 질문에서 헷갈린 지점

학습자는 `USTRUCT`와 `UENUM`을 보고 "클래스가 아닌 언리얼 오브젝트도 있다"고 판단했다. 또한 리플렉션은 실행 중 객체 정보를 조회하는 기능이라고 알고 있었기 때문에, 컴파일 시점은 리플렉션과 관계없다고 판단했다.

### 평가: 🟡

리플렉션의 런타임 사용 목적은 이해했지만 다음 범주가 섞였다.

- `UObject` 인스턴스
- 리플렉션에 등록된 타입
- 언리얼 프로젝트에서 사용하는 일반 C++ 객체

### 개념 교정

`UObject` 인스턴스에는 반드시 대응하는 `UClass` 정보가 있다. 반면 `USTRUCT`와 `UENUM`은 리플렉션 대상이지만 `UObject`는 아니다.

| 선언 | 대상 | 리플렉션 정보 | `UObject`인가? |
| --- | --- | --- | --- |
| `UCLASS()` | `UObject` 계열 클래스 | `UClass` | 예 |
| `USTRUCT()` | 구조체 타입 | `UScriptStruct` | 아니요 |
| `UENUM()` | 열거형 타입 | `UEnum` | 아니요 |

언리얼 리플렉션은 두 시점에 걸쳐 동작한다.

```text
소스 코드의 UCLASS·UPROPERTY·UFUNCTION
        ↓
빌드 시점: UHT가 타입 선언을 분석하고 지원 코드·메타데이터 생성
        ↓
C++ 컴파일과 링크
        ↓
실행 시점: 생성된 메타데이터로 객체의 타입·프로퍼티·함수 조회
```

시험의 "컴파일 시간에 객체 정보를 조회한다"는 표현은 엄밀하지 않다. 다음처럼 표현하는 것이 정확하다.

> 빌드 과정에서 UHT가 객체 타입의 선언을 분석하고, 런타임 리플렉션에 필요한 메타데이터와 등록 코드를 생성한다.

## 오답 분석 2 - 언리얼 인터페이스

### 질문에서 헷갈린 지점

학습자는 `U`로 시작하는 리플렉션 등록 클래스와 `I`로 시작하는 실제 인터페이스 클래스를 혼동했다. 또한 일반 C++ 인터페이스에 대한 기억을 바탕으로 인터페이스에는 순수 가상 함수만 선언할 수 있다고 판단했다.

### 평가: 🟡

인터페이스의 목적은 알고 있었지만 언리얼 인터페이스의 이중 클래스 구조와 기본 구현 가능 여부를 보완해야 했다.

### 개념 교정

```cpp
UINTERFACE()
class UInteractable : public UInterface
{
    GENERATED_BODY()
};

class IInteractable
{
    GENERATED_BODY()

public:
    virtual void Open() = 0;
    virtual bool CanInteract() const;
    FString GetInteractionName() const;
};
```

- `UInteractable`: 리플렉션 등록을 위한 빈 클래스
- `IInteractable`: 실제 인터페이스 함수를 담고 구현 클래스가 상속하는 클래스
- `ADoor`: `IInteractable`을 실제로 구현하는 클래스이며 이름이 `I`로 시작할 필요는 없음

인터페이스는 순수 가상 함수만으로 제한되지 않는다.

- 순수 가상 함수: 구현 클래스가 반드시 재정의
- 기본 구현이 있는 가상 함수: 필요할 때만 재정의
- 비가상 유틸리티 함수: 인터페이스의 구현을 그대로 상속

`IInteractable`의 함수는 헤더에서 인라인으로 정의하거나 인터페이스의 `.cpp` 파일에서 정의할 수 있다. 선언만 있고 정의가 없는 함수를 호출하면 일반 C++과 마찬가지로 링크 오류가 발생할 수 있다.

## 오답 분석 3 - 가비지 컬렉션

### 질문에서 헷갈린 지점

학습자는 문제에서 묻는 방향을 반대로 읽어, 가비지 컬렉션으로부터 유지되는 객체를 선택했다.

### 평가: 🟢 - 개념 통과, 문항 독해 주의

`UObject`가 가비지 컬렉션의 관리 대상이라는 말과 현재 수집 가능한 상태라는 말은 다르다. 가비지 컬렉터는 루트에서 추적 가능한 강한 참조를 따라가며 도달 가능한 객체를 유지하고, 도달할 수 없는 객체를 수집한다.

```cpp
UPROPERTY()
TObjectPtr<UItem> Item;
```

이 프로퍼티를 소유한 객체가 도달 가능한 상태라면 `Item`도 추적되므로 유지된다.

```cpp
UItem* Item = NewObject<UItem>();
```

`NewObject`는 객체를 생성할 뿐 영구적인 생존을 보장하지 않는다. 생성된 객체를 `UPROPERTY`, 엔진이 추적하는 컨테이너, 강한 객체 참조 또는 루트 등으로 보관하지 않으면 이후 가비지 컬렉션에서 수집될 수 있다.

### 교정된 한 문장

> `UObject`는 모두 GC 관리 대상이지만, 현재 수집되는 것은 루트에서 추적 가능한 강한 참조로 도달할 수 없는 객체다.

## 최종 30초 답변

> 언리얼 리플렉션은 `UCLASS`, `UPROPERTY`, `UFUNCTION` 등으로 표시한 타입 선언을 UHT가 빌드 중 분석해 메타데이터와 등록 코드를 만들고, 실행 중에는 그 정보를 이용해 객체의 타입과 멤버를 조회하는 구조입니다. `USTRUCT`와 `UENUM`도 리플렉션 대상이지만 `UObject`는 아닙니다. 언리얼 인터페이스는 리플렉션용 `U` 클래스와 실제 함수 선언을 가진 `I` 클래스로 나뉘며, 순수 가상 함수뿐 아니라 기본 구현 함수도 가질 수 있습니다. 가비지 컬렉터는 `UObject` 참조 그래프를 따라 도달 가능한 객체를 유지하고, 추적되지 않는 객체를 수집합니다.

## 복습 질문

1. UHT가 빌드 시점에 하는 일과 런타임 리플렉션이 하는 일은 어떻게 다른가요?
2. `UObject`, `UClass`, `USTRUCT`, `UScriptStruct`는 각각 무엇인가요?
3. 언리얼 인터페이스가 `U` 클래스와 `I` 클래스로 나뉘는 이유는 무엇인가요?
4. 언리얼 인터페이스에 기본 구현을 가진 함수를 작성하려면 어떻게 해야 하나요?
5. `NewObject`로 만든 객체가 항상 다음 GC까지 살아 있는 것은 아닌 이유가 무엇인가요?

## 참고

- [Epic Games - Reflection System](https://dev.epicgames.com/documentation/unreal-engine/reflection-system-in-unreal-engine)
- [Epic Games - Unreal Header Tool](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-header-tool-for-unreal-engine)
- [Epic Games - Interfaces](https://dev.epicgames.com/documentation/en-us/unreal-engine/interfaces-in-unreal-engine)
- [Epic Games - Objects](https://dev.epicgames.com/documentation/unreal-engine/objects-in-unreal-engine)
