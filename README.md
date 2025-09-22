---

# 🏃 [내일배움캠프 언리얼엔진] 캐릭터에 생명을! 입력과 액션 바인딩

> ## **학습 키워드**
>
> *   `SetupPlayerInputComponent`: 입력 액션(IA)과 캐릭터의 실제 동작 함수를 연결하는 핵심 함수.
>   
> *   `Action Binding`: "스페이스바를 누르면 점프 함수를 호출한다"와 같이 입력과 함수를 묶는 과정.
>   
> *   `FInputActionValue`: Enhanced Input System을 통해 전달되는 입력 값(Vector2D, bool 등)을 담는 구조체.
>   
> *   `AddMovementInput`: 캐릭터의 이동을 처리하는 내장 함수.
>   
> *   `CharacterMovementComponent`: 캐릭터의 속도, 점프력 등 모든 이동 관련 속성을 관리하는 컴포넌트.
>   
> *   `UFUNCTION()`: 입력 바인딩 함수가 엔진에 인식되도록 등록하는 필수 매크로.

<br>

> **입력 처리 흐름 요약**
> 1.  **`PlayerController`**: 키보드/마우스 입력을 감지하고, 연결된 `IMC`를 통해 어떤 `Input Action(IA)`이 발생했는지 파악합니다.
>
> 2.  **`Character`**: `SetupPlayerInputComponent` 함수 내에서, 각 `IA`가 발생했을 때 어떤 C++ 함수를 실행할지 미리 **'바인딩(Binding)'** 해둡니다.
>
> 3.  **실행**: 플레이어가 키를 누르면, 컨트롤러가 이를 감지하고, 캐릭터에 바인딩된 함수가 호출되어 실제 움직임이 일어납니다.

### 1️⃣ **캐릭터에 액션 바인딩 함수 선언 및 구현**

`MyCharacter` 클래스에서 각 입력에 대응하는 함수들을 선언하고, `SetupPlayerInputComponent` 함수에서 이들을 `IA`와 연결합니다.

*   **`MyCharacter.h` (헤더 파일)**
    ```cpp
    #pragma once
    
    #include "CoreMinimal.h"
    #include "GameFramework/Character.h"
    #include "MyCharacter.generated.h"
    
    // ... 전방 선언 ...
    struct FInputActionValue; // Enhanced Input의 입력 값을 담는 구조체
    
    UCLASS()
    class MYPROJECT_API AMyCharacter : public ACharacter
    {
    	GENERATED_BODY()
    
    public:
    	AMyCharacter();
    
    protected:
        // ... 컴포넌트 변수 ...
    
    	// 입력 바인딩을 설정하는 핵심 함수
    	virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;
    	
    	// 각 IA에 바인딩될 함수들
    	UFUNCTION() void Move(const FInputActionValue& Value);

    	UFUNCTION() void StartJump(const FInputActionValue& Value);
    	UFUNCTION() void StopJump(const FInputActionValue& Value);
    
    // ... (Look, Sprint 액션도 동일한 방식으로 선언) ...
    
    };
    ```

*   **`MyCharacter.cpp` (소스 파일)**
    ```cpp
    #include "MyCharacter.h"
    #include "MyPlayerController.h" // 컨트롤러에 정의된 IA 변수에 접근하기 위해 포함
    #include "EnhancedInputComponent.h"
    // ... 기타 헤더 ...
    
    void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
    {
    	Super::SetupPlayerInputComponent(PlayerInputComponent);
    
    	// PlayerInputComponent를 UEnhancedInputComponent로 캐스팅합니다.
    	if (UEnhancedInputComponent* EnhancedInput = Cast<UEnhancedInputComponent>(PlayerInputComponent))
    	{
    		// 현재 이 캐릭터를 조종하는 PlayerController를 가져옵니다.
    		if (AMyPlayerController* PlayerController = Cast<AMyPlayerController>(GetController()))
    		{
    			// MoveAction이 할당되어 있다면, Move 함수와 바인딩합니다.
    			if (PlayerController->MoveAction)
    			{
    				EnhancedInput->BindAction(PlayerController->MoveAction, ETriggerEvent::Triggered, this, &AMyCharacter::Move);
    			}
    
    			// JumpAction이 할당되어 있다면, StartJump/StopJump 함수와 바인딩합니다.
    			if (PlayerController->JumpAction)
    			{
    				EnhancedInput->BindAction(PlayerController->JumpAction, ETriggerEvent::Started, this, &AMyCharacter::StartJump);
    				EnhancedInput->BindAction(PlayerController->JumpAction, ETriggerEvent::Completed, this, &AMyCharacter::StopJump);
    			}
    
    			// ... (Look, Sprint 액션도 동일한 방식으로 바인딩) ...
    		}
    	}
    }
    
    // ... 각 함수의 구체적인 구현은 아래에서 계속 ...
    ```
> #### 💡 **학습 TIP: `BindAction` 함수 파헤치기**
>
> `BindAction(InputAction, TriggerEvent, Object, Function)`
> 
> *   `InputAction`: `PlayerController`에 선언해둔 `IA` 변수를 전달합니다.
> *   
> *   `TriggerEvent`: 어떤 시점에 함수를 호출할지 결정합니다.
> *
> *   `Triggered`(키를 누르고 있는 동안 매번),
> *
> *   `Started`(처음 누른 순간),
> *
> *   `Completed`(키를 뗀 순간) 등이 있습니다.
> *   
> *   `Object` & `Function`: '어떤 객체(`this`)'의 '어떤 함수(`&AMyCharacter::Move`)'를 호출할지 지정합니다.

<br>

---

## **2. 캐릭터 이동 및 시점 회전 구현하기**

---

### 1️⃣ **Move & Look 함수 구현**

바인딩된 함수 내부에 실제 동작 코드를 작성합니다. `ACharacter` 클래스에는 이동과 관련된 유용한 함수들이 이미 내장되어 있어 쉽게 구현할 수 있습니다.

*   **`MyCharacter.cpp` (소스 파일)**
    ```cpp
    void AMyCharacter::Move(const FInputActionValue& Value)
    {
        if (!Controller) return; // 컨트롤러가 없으면 이동 불가
    
        // IA_Move는 Axis2D이므로, FVector2D 형태로 입력 값을 가져옵니다.
        const FVector2D MoveInput = Value.Get<FVector2D>();
    
        // 앞/뒤 이동 (X축 입력)
        if (!FMath::IsNearlyZero(MoveInput.X))
        {
            AddMovementInput(GetActorForwardVector(), MoveInput.X);
        }
    
        // 좌/우 이동 (Y축 입력)
        if (!FMath::IsNearlyZero(MoveInput.Y))
        {
            AddMovementInput(GetActorRightVector(), MoveInput.Y);
        }
    }
    
    ```

### 2️⃣ **Jump 함수 구현**

점프 역시 `ACharacter`에 내장된 `Jump()`와 `StopJumping()` 함수를 호출하기만 하면 됩니다.

*   **`MyCharacter.cpp` (소스 파일)**
    ```cpp
    void AMyCharacter::StartJump(const FInputActionValue& Value)
    {
        // ACharacter에 내장된 점프 함수 호출
        Jump();
    }
    
    void AMyCharacter::StopJump(const FInputActionValue& Value)
    {
        // ACharacter에 내장된 점프 중지 함수 호출
        StopJumping();
    }
    ```

<br>

---

## **3. 스프린트(달리기) 기능 구현하기**

---

스프린트 기능은 `CharacterMovementComponent`의 `MaxWalkSpeed` 프로퍼티 값을 조절하여 구현합니다.

### 1️⃣ **관련 변수 추가 및 초기화**

*   **`MyCharacter.h` (헤더 파일)**
    ```cpp
    protected:
        // 에디터에서 쉽게 수정할 수 있도록 UPROPERTY로 노출
    	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Movement")
    	float WalkSpeed;
    
    	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Movement")
    	float SprintSpeed;
    ```

*   **`MyCharacter.cpp` (소스 파일)**
    ```cpp
    #include "GameFramework/CharacterMovementComponent.h"
    
    AMyCharacter::AMyCharacter()
    {
        // ... 이전 코드 ...
    
        // 기본 속도 값 설정
        WalkSpeed = 600.0f;
        SprintSpeed = 1000.0f;
    
        // 게임 시작 시 기본 속도를 MaxWalkSpeed에 적용
        GetCharacterMovement()->MaxWalkSpeed = WalkSpeed;
    }
    ```

### 2️⃣ **Sprint 함수 구현**

Shift 키를 누르면 `MaxWalkSpeed`를 `SprintSpeed`로, 키를 떼면 다시 `WalkSpeed`로 변경합니다.

*   **`MyCharacter.cpp` (소스 파일)**
    ```cpp
    void AMyCharacter::StartSprint(const FInputActionValue& Value)
    {
        // 이동 속도를 SprintSpeed로 변경
        GetCharacterMovement()->MaxWalkSpeed = SprintSpeed;
    }
    
    void AMyCharacter::StopSprint(const FInputActionValue& Value)
    {
        // 이동 속도를 다시 원래 WalkSpeed로 변경
        GetCharacterMovement()->MaxWalkSpeed = WalkSpeed;
    }
    ```

이제 모든 코드를 빌드하고 에디터에서 플레이하면, WASD로 이동하고, 마우스로 시점을 돌리고, 스페이스바로 점프하며, 

Shift 키를 눌러 달리는 기본적인 3인칭 캐릭터 조작이 완성된 것을 확인할 수 있습니다. 

---

#언리얼엔진 #캐릭터이동 #입력바인딩 #내일배움캠프
