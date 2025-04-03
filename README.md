# MyFactory - 언리얼 엔진 프로젝트 포트폴리오

## 프로젝트 개요

이 프로젝트는 언리얼 엔진을 사용하여 컨베이어 벨트 시스템과 로봇 팔을 이용한 가상 생산 라인을 시뮬레이션합니다. 주요 기능은 다음과 같습니다:

*   엔진 스포너(`BP_EngineSpawner`)가 주기적으로 엔진 객체(`BP_ConveyorEngine`)를 생성합니다.
*   생성된 엔진은 지정된 컨베이어 경로(`BP_ConveyorPath`)를 따라 이동합니다.
*   로봇 팔(`BP_RobotArm`)은 감지 영역(`BP_DetectionZone`)에 들어온 엔진을 감지하고 상호작용합니다.
*   생산 관리자(`BP_ProductionManager`)는 경로 끝에 도달한 완성된 엔진의 개수를 집계하고 UI(`WBP_ProductionCounter`)에 표시합니다.

아래는 프로젝트를 구성하는 주요 블루프린트 클래스들에 대한 설명입니다.
모든 블루프린트 클래스들은 Content/Blueprints/Conveyor 경로 내부에 존재합니다. 

---

## 주요 블루프린트 설명

### `BP_EngineSpawner` 액터
![Image](https://github.com/user-attachments/assets/700e38a2-475f-430e-b711-18e77d0d6581)
*   **핵심 기능:** 지정된 컨베이어 벨트 경로(`Target Conveyor Path`)의 시작 지점에 주기적으로 엔진 객체(`EngineClass`)를 생성(스폰)합니다.
*   **주요 작동 방식:**
    1.  게임 시작 시(`BeginPlay`), 대상 컨베이어 경로가 유효하고 **`Is Spawning` 플래그가 `True`**이면, 설정된 시간 간격(`Spawn Interval`)마다 엔진 스폰 로직을 실행하는 타이머를 활성화합니다.
        *   **참고:** `Is Spawning`의 **기본값은 `True`**이므로, 별도의 설정을 변경하지 않으면 스포너는 배치되자마자 **기본적으로 엔진 스폰을 시작**합니다. 스폰을 중지하려면 이 값을 `False`로 변경해야 합니다.
    2.  타이머에 의해 주기적으로 호출되는 `Spawn Engine` 함수는 다음을 수행합니다:
        *   컨베이어 경로의 시작 위치를 계산합니다.
        *   지정된 `EngineClass` 타입의 액터를 해당 위치에 스폰합니다.
        *   스폰된 엔진 액터의 `Initialize Engine` 함수를 호출하여, 이동해야 할 컨베이어 경로 정보를 전달하고 초기화합니다.
*   **핵심 설정:**
    *   `Target Conveyor Path`: 엔진이 따라갈 컨베이어 벨트 경로 지정.
    *   `EngineClass`: 스폰할 엔진의 종류(블루프린트 클래스) 지정.
    *   `Spawn Interval`: 엔진 스폰 주기 (초 단위).
    *   `Is Spawning`: 스포너 활성화 여부 제어 (**기본값 `True`, `True` 상태여야 스폰 시작**).

---

### `BP_ConveyorEngine` 액터
![Image](https://github.com/user-attachments/assets/940b8a92-be44-4071-99fb-1a284c301a61)
*   **주요 역할:** `BP_EngineSpawner`에 의해 스폰되어, 지정된 컨베이어 경로(`BP_ConveyorPath` 액터)를 따라 설정된 속도로 이동하는 '엔진' 객체를 나타냅니다. 경로의 끝에 도달하면 완료 처리를 수행하고 스스로 소멸합니다.
*   **세부 로직:**
    1.  **초기화 (`Initialize Engine` 함수):** 스포너로부터 컨베이어 경로(`ConveyorPath` - `BP_ConveyorPath` 액터 참조)와 시작 거리 정보를 받아 저장하고, 이동 상태(`IsMoving`)를 활성화하며 초기 위치를 업데이트합니다.
    2.  **매 프레임 이동 처리 (`Event Tick`):** `IsMoving` 상태일 때, `MoveSpeed`에 따라 `DistanceAlongSpline`(누적 이동 거리)을 업데이트하고 `Update Engine Position` 함수를 호출하여 실제 위치/회전을 갱신합니다. 경로 끝에 도달하면(`DistanceAlongSpline` > 스플라인 길이) `HasCompletedPath` 플래그를 설정하고 `BP_ProductionManager`에 완료 신호를 보낸 후, 잠시 뒤 스스로 파괴됩니다.
    3.  **위치 및 회전 업데이트 (`Update Engine Position` 함수):** 현재 `DistanceAlongSpline` 값에 해당하는 스플라인 상의 위치와 회전 값을 가져와 액터의 트랜스폼을 업데이트합니다. 이를 통해 컨베이어의 곡선 경로를 따라 자연스럽게 이동합니다.
*   **주요 변수:**
    *   `ConveyorPath` (**BP_ConveyorPath Object Reference**): 따라가야 할 컨베이어 경로 액터 참조.
    *   `Conveyor Spline` (Spline Component Ref): 실제 이동 경로 계산에 사용되는 스플라인.
    *   `DistanceAlongSpline` (Float): 스플라인 시작점으로부터 이동한 총 거리.
    *   `MoveSpeed` (Float): 이동 속도.
    *   `IsMoving` (Boolean): 현재 이동 중인지 여부.
    *   `HasCompletedPath` (Boolean): 경로 완료 처리 여부 플래그.
*   **상호작용:** `BP_EngineSpawner`로부터 생성/초기화되고, `BP_ProductionManager`에게 완료 상태를 알립니다.

---

### `BP_RobotArm` 액터
![Image](https://github.com/user-attachments/assets/c980fd03-01d2-43a2-a4ac-4f802e9b9619)
*   **주요 역할:** 컨베이어 벨트를 따라 이동하는 `BP_ConveyorEngine` 객체가 자신의 감지 영역(`BP_DetectionZone`)에 들어오면 이를 감지하고 상호작용하는 로봇 팔입니다. 엔진이 감지되면 로봇 팔은 해당 엔진의 특정 지점을 향해 부드럽게 회전하여 추적하고, 관련된 애니메이션을 재생합니다. 엔진이 영역을 벗어나면 로봇 팔은 초기 상태로 리셋됩니다.
*   **핵심 작동 흐름 및 상태 관리:**
    1.  **초기화 (`Event BeginPlay`):** 로봇 팔의 초기 회전 값을 `InitialRotation`에 저장합니다.
    2.  **유휴 상태 (Idle - `IsProcessing` is `False`):** 기본 상태. 추적 동작 없음.
    3.  **엔진 처리 시작 (`Process Engine` 함수):** 엔진 감지 시(외부 호출), 유휴 상태라면:
        *   감지된 엔진을 `CurrentEngine`으로 등록.
        *   `IsProcessing`, `HasValidTarget` 플래그를 `True`로 설정 (처리 중 상태).
        *   로봇 팔 위치(`Arm Position`)에 따라 추적할 엔진 접점(`Engine Target Point`) 결정.
        *   처리 애니메이션(`ProcessingAnimation`) 재생.
    4.  **엔진 추적 및 회전 (Processing - `IsProcessing` is `True`, `Event Tick`):** 매 프레임마다:
        *   `Engine Target Point`를 향하도록 목표 회전 계산 (수평 Yaw만 적용).
        *   `RInterp To`를 사용하여 현재 회전에서 목표 회전으로 부드럽게 보간 및 적용.
    5.  **엔진 처리 종료 및 리셋 (`Reset Robot` 함수 & `Start Reset Rotation` 이벤트):** 엔진이 영역 이탈 시(외부 호출), 해당 엔진이 `CurrentEngine`이면:
        *   상태 플래그(`IsProcessing`, `HasValidTarget`) 및 참조(`CurrentEngine`, `Engine Target Point`) 리셋.
        *   애니메이션 중지.
        *   `Start Reset Rotation` 이벤트 호출.
    6.  **회전 리셋 (`Start Reset Rotation` 이벤트):** 타임라인을 사용하여 1초 동안 로봇 팔 회전을 `InitialRotation` 값으로 부드럽게 되돌림. 완료 후 다시 유휴 상태.
*   **주요 변수:** `IsProcessing`, `CurrentEngine`, `InitialRotation`, `Engine Target Point`, `Arm Position`, `ArmEnd`, `Arm Component`, `ProcessingAnimation`, `HasValidTarget`.
*   **상호작용:** `BP_DetectionZone`에 의해 트리거되고, `BP_ConveyorEngine`과 상호작용합니다.

---

### `BP_DetectionZone` 액터
![Image](https://github.com/user-attachments/assets/42c103ba-64e8-417c-a2a0-cedbc96d6bae)
*   **주요 역할:** 레벨에 배치되어 특정 영역(`DetectionZone` 콜리전)을 감시합니다. `BP_ConveyorEngine` 객체가 이 영역에 들어오거나 나갈 때, 연결된 특정 `BP_RobotArm` 액터(`TargetRobotArm` 변수)에게 해당 엔진 정보를 전달하여 로봇 팔의 작동 시작 또는 중지(리셋)를 트리거합니다.
*   **세부 로직:**
    1.  **엔진 감지 시작 (`On Component Begin Overlap`):** 영역에 들어온 액터가 `BP_ConveyorEngine`이고, `TargetRobotArm`이 유효하면, 해당 로봇 팔의 `Process Engine` 함수를 호출하며 엔진 정보를 전달합니다.
    2.  **엔진 감지 종료 (`On Component End Overlap`):** 영역을 벗어난 액터가 `BP_ConveyorEngine`이고, `TargetRobotArm`이 유효하면, 해당 로봇 팔의 `Reset Robot` 함수를 호출하며 엔진 정보를 전달합니다.
*   **주요 변수:**
    *   `TargetRobotArm` (BP_RobotArm Ref, **Instance Editable**): 이 감지 구역과 연결될 특정 로봇 팔 참조 (레벨 편집 시 지정).
*   **상호작용:** `BP_ConveyorEngine`의 오버랩 이벤트를 감지하고, 연결된 `BP_RobotArm`의 함수를 호출하여 제어합니다.

---

### `BP_ProductionManager` 액터 & `WBP_ProductionCounter` 위젯
![Image](https://github.com/user-attachments/assets/5fc6d07c-fef0-48ea-b9c7-fdb9d2e68ab3)
*   **개요:** 완성된 엔진의 개수를 집계하고 UI에 실시간으로 표시하는 시스템입니다. 매니저가 데이터 관리 및 업데이트 트리거를, 위젯이 UI 표시를 담당합니다.
*   **`BP_ProductionManager` 액터:**
    *   **역할:** 완성품 개수(`CompletedItems`) 추적, `WBP_ProductionCounter` UI 관리 및 업데이트 지시.
    *   **로직:**
        *   `BeginPlay`: `WBP_ProductionCounter` 위젯 생성, 뷰포트에 추가, `ProductionUI` 변수에 참조 저장, 초기 UI 업데이트 호출.
        *   `Complete One Item`: 외부(`BP_ConveyorEngine`) 호출 시 `CompletedItems` 1 증가, `Update Production UI` 호출.
        *   `Update Production UI`: `ProductionUI` 위젯의 `Update Counter` 함수 호출하며 `CompletedItems` 값 전달.
    *   **변수:** `CompletedItems` (Integer), `ProductionUI` (Widget Ref).
*   **`WBP_ProductionCounter` 위젯 블루프린트:**
    *   **역할:** `BP_ProductionManager`로부터 받은 개수 정보를 화면 텍스트로 표시.
    *   **로직:**
        *   `Update Counter`: 입력받은 개수(`Value`)를 사용하여 `Format Text`("생산 현황: {Value}개")로 문자열 생성 후, 위젯 내 텍스트 블록에 설정하여 화면 갱신.
*   **상호작용 흐름:** 엔진 완료 → `BP_ProductionManager`.`Complete One Item` → `CompletedItems` 증가 → `BP_ProductionManager`.`Update Production UI` → `WBP_ProductionCounter`.`Update Counter` → UI 텍스트 갱신.

---
