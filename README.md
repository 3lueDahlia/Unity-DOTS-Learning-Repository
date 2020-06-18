# Unity-DOTS-Learning-Repository
개인적인 공부를 정리하기 위한 저장소.
DOTS - Data Oriented Tech Stack - 데이터 지향 기술 스택

---

## DOTS
### 주요 기능
- **ECS (Entitiy Component System)**
  - 데이터 지향적 접근 방식을 이용한 새로운 컴포넌트 시스템.
  메모리에서 컴포넌트 세트가 완벽하게 동일한 모든 엔티티를 그룹화 시켜 관리. 이 컴포넌트 세트를 원형(Archetype)이라 부른다.
  - ECS는 16K의 청크 단위로 메모리를 할당한다. 각 청크에는 하나의 원형에 속한 엔티티 컴포넌트 데이터만 포함된다.
  - 컴포넌트를 찾을 때는, '어떤 컴포넌트가 있는 모든 엔티티를 대상으로 작업을 실행할 것' 이라고 정적으로 선언해야 한다. 이는 **컴포넌트 검색 쿼리**와 일치하는 모든 원형을 찾으면 된다. 이 프로세스는 여러 잡(Job)으로 분할되어 코어를 최대로 활용하며 실행된다.
  
- **Job System**
  - 멀티 코어를 **쉽고**, **안전하게**, 그러면서도 **최대로 활용**하기 위한 새로운 멀티 쓰레딩 시스템
  
- **Burst Compiler**
  - 개별 플랫폼과 하드웨어에 맞춰 극한으로 최적화 되어 실행되게 하는 컴파일러
  - 기존 CLRVM이 IL 코드를 해석하고 실행하는 속도는 Native에 비해 느릴 수 밖에 없기에 LLVM 프로젝트를 확장하여 개발된 기능. 대상 플랫폼에 맞춰 자동으로 최적화된 코드로 변환해준다.
  - ECS 기반의 코드면 큰 수정 없이 사용 가능하다.


| DOTS 구성요소 | 담당 기능 |
| :---: | :---: |
| Entity Component System | 메모리 설계 최적화 |
| Job System | 멀티 쓰레딩 기능 강화 |
| Burst Compiler | 컴파일 성능 최적화 |

---
## Entity Component System
### 문법
#### ComponentData
    컴포넌트에 사용될 정보를 저장하는 구조체

  ##### IComponentData
  - 범용 컴포넌트를 구현하기 위한 인터페이스.
    ```csharp
    [GenerateAuthoringComponent]
    public struct ComponentData : IComponentData
    {
        public float fComponentValue;
    }
    ```
  - 인터페이스를 포함하는 구조체여야 하며, 관리되지 않거나?(Unmanaged), Blittable 유형만 포함할 수 있다.
    - C#에서 정의된 Blittable 타입
    - bool
    - char
    - (고정된 크기의 문자 버퍼)
    - BlobAssetReference<T> (블롭 데이터 구조에 대한 참조)
    - 고정 배열 (불안전 컨텍스트)
    - 관리되지 않거나?(Unmanaged), 블리터블 필드를 포함한 구조체
  - **GenerateAuthoringComponent :** ECS 컴포넌트는 MonoBehaviour를 상속받는 존재가 아니기에 인스펙터에 노출 시킬수가 없다. 그래서 사용하는게 해당 속성인데, 구조체에 부여하면 엔티티의 인스펙터에 노출시킬 수 있게 되어 초기 값을 지정할 수 있게 된다.   
  유니티에서 자동으로 public 필드를 포함하는 MonoBehaviour 클래스를 생성하여 인스펙터에 노출시켜준다. 

  ##### ComponentDataProxy
  - ~~범용 컴포넌트 구조체를 인스펙터에 노출 시키기위해 사용하는 프록시 클래스~~
  > **Deprecated**. use GameObject-to-Entity Conversion workflow instead.
  ```csharp
  [DisallowMultipleComponent]
  public class ComponentProxy : ComponentDataProxy<ComponentDataStruct> { }
```

#### SystemBase
    ComponentData를 관리하는 매니저 형태의 클래스

  ##### ComponentSystem
  - 메인 쓰레드만 이용하는 컴포넌트 관리 시스템
  -구현 정보
    - ForEach delegate를 이용한 반복 처리
      ```csharp
      public class AwesomeComponentSystem : ComponentSystem 
      {
        protected override void OnUpdate() 
        {
          Entities.ForEach((ref ComponentDataA componentDataA, ref ComponentDataB componentDataB) => 
          {
            /* do someting */
          }); 
        }
      }
      ```
  
  ##### [JobComponentSystem](https://github.com/3lueDahlia/Unity-DOTS-Learning-Repository/blob/master/README.md#JobComponentSystem)

#### Entity
- 1개 이상의 컴포넌트를 지닌 오브젝트
  - 게임 오브젝트를 엔티티화 시키기 위해선, **Convert To Entity** 라는 컴포넌트를 할당해야 한다.
- Component 시스템 내에서 특정 엔티티들을 가져오기 위한 Entity Query를 람다식으로 지원한다.
- 작업의 진행은 Run, Schedule, ScheduleParallel 로 나뉜다.
  - **Run :** 메인 스레드에서 즉시 실행한다.
  - **Schedule :** 단일 스레드에 작업을 예약한다
  - **ScheduleParallel :** 병렬 스레드에 작업을 예약한다
- ex)
  ```csharp
  Entities.WithAll<LocalToWorld>()
    .WithAny<Rotation, Translation, Scale>()
    .WithNone<LocalToParent>()
    .ForEach((ref ComponentDataA outputDataA, in ComponentDataB inputDataB) =>
    {
        /* do some work */
    })
    .Schedule();
  ```
  - 이 경우엔 LocalToWorld, ComponentDataA, ComponentDataB를 포함하고, Rotation, Translation, Scale 중 하나라도 포함하며, LocalToParent는 없는 엔티티를 모두 가져오고, 작업은 단일 쓰레드에 예약한다.
  
##### Entitiy Query


---
## Job System
### JobComponentSystem
- 멀티 쓰레딩을 활용하는 컴포넌트 관리 시스템
- 구현 정보
  - JobForEach
  
  ```csharp
  public class RotationSpeedSystem : JobComponentSystem 
  {
    // [BurstCompile] attribute를 사용시, job 코드 컴파일 시 Burst 컴파일러가 사용된다. 
    [BurstCompile]
    struct RotationSpeedJob : IJobForEach<RotationQuaternion, RotationSpeed> 
    {
      public float DeltaTime;
      public void Execute(ref RotationQuaternion rotationQuaternion, [ReadOnly] ref RotationSpeed rotSpeed) 
      { // [ReadOnly] attribute를 사용해서 job이 대상 컴포넌트 데이터를 읽기만 하고 쓰지 않는다고 명시한다. 
        // RotationSpeed를 속도 값으로 취해서 up 벡터 방향으로 enitity를 회전시키는 로직이다.
        rotationQuaternion.Value = math.mul(math.normalize(rotationQuaternion.Value), 
          quaternion.AxisAngle(math.up(), rotSpeed.RadiansPerSecond * DeltaTime)); 
      } 
    }
    
    // OnUpdate는 메인 쓰레드에서 실행됨
    // 'Rotation'에서 읽거/쓰기로 또는 'RotationSpeed'에서 읽기 이전에 예약된 모든 작업은 자동으로 '입력 종속성'에 포함된다.
    protected override JobHandle OnUpdate(JobHandle inputDependencies) 
    {
      var job = new RotationSpeedJob() 
      {
        DeltaTime = Time.deltaTime 
      }; 

      return job.Schedule(this, inputDependencies); 
    }
  }
  ```
  
  - IJobForEachWithEntity
  - IJobChunk
  - 수동 반복(Manual iteration)

---
## Burst Compiler
