# Unity-DOTS-Learning-Repository
개인적인 공부를 정리하기 위한 저장소.
DOTS - Data Oriented Tech Stack - 데이터 지향 기술 스택.

아래에 서술된 내용은 국내,외 여러 레퍼런스를 참고하여 정리한 것입니다.
현재 사용되지 않는, 대체되려 하는 기능이 포함될 수 있고 틀린 정보가 있을 수 있습니다.

---

## DOTS
### 주요 기능
- **[ECS (Entitiy Component System)](https://github.com/3lueDahlia/Unity-DOTS-Learning-Repository/blob/master/README.md#entity-component-system)**
  - 데이터 지향적 접근 방식을 이용한 새로운 컴포넌트 시스템.
  메모리에서 컴포넌트 세트가 완벽하게 동일한 모든 엔티티를 그룹화 시켜 관리. 이 컴포넌트 세트를 원형(Archetype)이라 부른다.
  - ECS는 16K의 청크 단위로 메모리를 할당한다. 각 청크에는 하나의 원형에 속한 엔티티 컴포넌트 데이터만 포함된다.
  - 컴포넌트를 찾을 때는, '어떤 컴포넌트가 있는 모든 엔티티를 대상으로 작업을 실행할 것' 이라고 정적으로 선언해야 한다. 이는 **컴포넌트 검색 쿼리**와 일치하는 모든 원형을 찾으면 된다. 이 프로세스는 여러 잡(Job)으로 분할되어 코어를 최대로 활용하며 실행된다.
  
- **[Job System](https://github.com/3lueDahlia/Unity-DOTS-Learning-Repository/blob/master/README.md#job-system)**
  - 멀티 코어를 **쉽고**, **안전하게**, 그러면서도 **최대로 활용**하기 위한 새로운 멀티 쓰레딩 시스템
  
- **[Burst Compiler](https://github.com/3lueDahlia/Unity-DOTS-Learning-Repository/blob/master/README.md#burst-compiler)**
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
  - 인터페이스를 포함하는 구조체여야 하며, 비관리형, Blittable 유형만 포함할 수 있다.
    - C#에서 정의된 Blittable 타입
    - bool
    - char
    - (고정된 크기의 문자 버퍼)
    - BlobAssetReference<T> (블롭 데이터 구조에 대한 참조)
    - 고정 배열 (불안전 컨텍스트)
    - 비관리형, 블리터블 필드를 포함한 구조체
  - **GenerateAuthoringComponent :** ECS 컴포넌트는 MonoBehaviour를 상속받는 존재가 아니기에 인스펙터에 노출 시킬수가 없다. 그래서 사용하는게 해당 속성인데, 구조체에 부여하면 엔티티의 인스펙터에 노출시킬 수 있게 되어 초기 값을 지정할 수 있게 된다.   
  유니티에서 자동으로 public 필드를 포함하는 MonoBehaviour 클래스를 생성하여 인스펙터에 노출시켜준다. 

  ##### ComponentDataProxy
  - ~~범용 컴포넌트 구조체를 인스펙터에 노출 시키기위해 사용하는 프록시 클래스~~
  > **Deprecated**. use GameObject-to-Entity Conversion workflow instead. *대체용으로 GenerateAuthoringComponent를 사용하는 듯 하다.*

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
  
  ##### [JobComponentSystem](https://github.com/3lueDahlia/Unity-DOTS-Learning-Repository/blob/master/README.md#jobcomponentsystem-1)
  - 멀티 쓰레딩을 활용하는 컴포넌트 관리 시스템. 아래의 Job System에서 상세 서술.

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
- EntityQuery는 시스템이 관련 청크나 엔티티 를 처리하기 위해 아키타입에 포함해야 하는 [컴포넌트 타입의 묶음]을 정의한다.   
아키타입은 추가 컴포넌트를 가질 수 있지만 최소한 EntityQuery에 의해 정의 된 컴포넌트를 가져야한다.   
특정 엔티티의 컴포넌트가 포함 된 아키타입도 제외 할 수 있다.

시스템은 처리 할 엔티티가 포함 된 각 청크에 대해 Execute() 함수를 한 번씩 호출한다. 그 때, 각 청크 내부의 데이터를 엔티티 별로 처리 할 수 있다.
- 작업 구현 단계
  - EntityQuery를 작성하여 처리하려는 엔티티를 식별한다,
  - Job 이 해당 컴포넌트에 대한 읽기/쓰기 여부를 지정하여 Job 이 직접 접근하는 컴포넌트의 타입을 식별하기 위해 ArchetypeChunkComponentType 오브젝트에 대한 필드를 포함하여 Job 구조체를 정의한다.
  - System 의 OnUpdate () 함수에서 Job 구조체를 인스턴스화하고 작업을 예약한다.
  - Execute() 함수에서 Job이 읽거나 쓰는 컴포넌트의 NativeArray 인스턴스를 가져오고 마지막으로 현재 청크를 반복하여 원하는 작업을 수행합니다.

    > 간단한 쿼리의 경우, JobComponentSystem.GetEntityQuery() 에 컴포넌트 타입을 전달하여 사용할 수 있다.
    ```csharp
    public class RotationSpeedSystem : JobComponentSystem
    {
       private EntityQuery m_Group;

       protected override void OnCreate()
       {
           m_Group = GetEntityQuery(typeof(RotationQuaternion), ComponentType.ReadOnly<RotationSpeed>());
       }
       //…
    }
    ```
    
    > 복잡한 상황의 경우  EntityQueryDesc 를 사용할 수 있다. EntityQueryDesc는 컴포넌트 타입의 유연한 지정을 위한 쿼리 메커니즘을 제공한다.
    ```csharp
    public class RotationSpeedSystem : JobComponentSystem
    {
      protected override void OnCreate()
      {
         var query = new EntityQueryDesc
         {
             None = new ComponentType[]{ typeof(Frozen) },
             All = new ComponentType[]{ typeof(RotationQuaternion), ComponentType.ReadOnly<RotationSpeed>() }
             /*
             All =이 배열의 모든 Component 타입은 Archetype에 존재해야 한다.
             Any =이 배열의 Component 타입 중 하나 이상이 Archetype에 존재해야 한다.
             None =이 배열의 어떤 Component 타입도 Archetype에 존재하지 말아야 한다.
             
             RotationQuaternion 및 RotationSpeed 이라는 Component 가 포함 된 Archetype이 포함되지만 Frozen 이라는 Component가 포함 된 Archetype 은 제외됩니다. */
         }};
         m_Group = GetEntityQuery(query);
         // m_Group = GetEntityQuery(new EntityQueryDesc[] {query0, query1}); 를 이용해서 여러 쿼리를 겹치게 할 수도 있다.
      }
    }
    
    ```
    컴포넌트의 값이 변경되었을 때만 엔티티를 업데이트해야 하는 경우 쿼리의 **변경 필터**에 대상 컴포넌트를 추가할 수 있다.   
    최대 2개까지 확인할 수 있기에, 더 많은 비교가 필요하거나 EntityQuery를 사용하지 않는 경우 [수동 반복](https://github.com/3lueDahlia/Unity-DOTS-Learning-Repository/blob/master/README.md#job-system)으로 확인할 수 있다.(Job System 하단에 Manual Iteration에 대한 상세 설명)
    
    ```chsarp
    class RotateComponentSystem : JobComponentSystem
    {
      EntityQuery m_Group;
      protected override void OnCreate()
      {
         m_Group = GetEntityQuery(typeof(Output), 
                                     ComponentType.ReadOnly<InputA>(), 
                                     ComponentType.ReadOnly<InputB>());
         m_Group.SetFilterChanged(new ComponentType{ typeof(InputA), typeof(InputB)});
      }
    }
    ```


---
## Job System
### JobComponentSystem
- 멀티 쓰레딩을 활용하는 컴포넌트 관리 시스템
- 구현 정보
  - JobForEach   
    ~~작업이 실행되면 ECS 프레임 워크는 필요한 컴포넌트가 있는 모든 엔티티를 찾고 각 엔티티에 대한 Execute() 함수를 호출한다.~~   
    ~~데이터는 메모리에 배치 된 순서대로 처리되고 작업이 병렬로 실행되므로 IJobForEach 에 단순성과 효율성이 결합된다.~~
    > Deprecated Use Entities.ForEach or IJobChunk to Schedule
  
  - IJobForEachWithEntity
    ~~IJobForEach와 비슷하게 동작하지만, 차이점은 현재 엔티티의 Entity 오브젝트와 인덱스, 컴포넌트 배열을 제공한다.~~
    ~~엔티티 오브젝트를 사용하여 EntityCommandBuffer에 명령을 추가할 수 있게 된다.~~
    ~~예를 들면, 해당 엔티티에서 컴포넌트를 추가 또는 제거하거나, 엔티티를 파괴하는 명령을 추가하여 경쟁 조건을 피하기 위해 특정 작업 도중에 직접 수행 할 수 없는 작업을 수행 시킬 수 있게된다.~~
    > Deprecated Use Entities.ForEach or IJobChunk to Schedule
  
  - IJobChunk
    IJobChunk 인터페이스를 구현하면, Entity 단위가 아니라 Chunk 메모리 블록 단위로 데이터의 반복 처리가 가능하다.
    
    시스템이 Execute() 메소드에 전달하는 청크 내부의 컴포넌트 배열에 접근하려면 Job이 읽기/쓰기가 가능한 컴포넌트 타입에 대한 객체인 ArchetypeChunkComponentType을 작성해야 한다. 객체를 사용하면 엔티티의 컴포넌트에 접근 할 수있는 NativeArray 인스턴스를 얻을 수 있다.   
    
    ```chsarp
    [BurstCompile]
    struct RotationSpeedJob : IJobChunk
    {
       public float DeltaTime;
       public ArchetypeChunkComponentType<RotationQuaternion> RotationType;
       [ReadOnly] public ArchetypeChunkComponentType<RotationSpeed> RotationSpeedType;
        // 대상 컴포넌트 배열에 접근하기 위한 ArchetypeChunkComponentType 객체

       public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
       {
          var chunkRotations = chunk.GetNativeArray(RotationType);
          var chunkRotationSpeeds = chunk.GetNativeArray(RotationSpeedType);

          for (var i = 0; i < chunk.Count; i++)
          {  //chunk.Count는 현재 청크에 저장된 엔티티의 개수.
             var rotation = chunkRotations[i];
             var rotationSpeed = chunkRotationSpeeds[i];

             // RotationSpeed를 속도 값으로 math.up() 벡터 방향으로 enitity를 회전시키는 로직.
             chunkRotations[i] = new RotationQuaternion
             {
                 Value = math.mul(math.normalize(rotation.Value),
                     quaternion.AxisAngle(math.up(), rotationSpeed.RadiansPerSecond * DeltaTime))
             };
          }
       }
    }
    ```
    컴포넌트의 값이 변경되었을 때만 엔티티를 업데이트해야 하는 경우 수동으로 비교하여 처리할 수 있다.   
    ArchetypeChunk.DidChange()함수를 사용하여 컴포넌트의 청크 변경 버전을 시스템의 LastSystemVersion과 비교한다.   
    DidChange() 함수가 false를 반환하면 시스템을 마지막으로 실행 한 이후 해당 엔티티의 컴포넌트가 변경되지 않았으므로 현재 청크를 건너뛰는 기능의 구현이 가능해진다.
    ```chsarp
    [BurstCompile]
    struct UpdateJob : IJobChunk
    {
       public ArchetypeChunkComponentType<InputA> InputAType;
       public ArchetypeChunkComponentType<InputB> InputBType;
       [ReadOnly] public ArchetypeChunkComponentType<Output> OutputType;
       public uint LastSystemVersion;
       // struct 필드를 사용하여 시스템의 LastSystemVersion을 작업으로 전달해야합니다.

       public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
       {
           var inputAChanged = chunk.DidChange(InputAType, LastSystemVersion);
           var inputBChanged = chunk.DidChange(InputBType, LastSystemVersion);
           if (!(inputAChanged || inputBChanged))
               return;
          //...
    }
    
    // 작업을 예약(Schedule) 하기 전에 LastSystemVersion을 지정해야 한다.
    public class  UpdateSystem : JobComponentSystem
    {
      protected override JobHandle OnUpdate(JobHandle inputDeps)
      {
        var job = new UpdateJob()
        {
             LastSystemVersion = this.LastSystemVersion,
             //… initialize other fields
        }
      }
    }
    ```
    
    IJobChunk 작업을 실행하려면 Job 구조체의 인스턴스를 작성하고 구조체 필드를 설정 한 후 작업을 예약해야한다.   
    JobComponentSystem의 OnUpdate() 함수에서 이 작업을 수행하면 시스템은 작업이 매 프레임마다 실행되도록 예약한다.
    ```csharp
    // OnUpdate는 매 프레임마다 호출된다.
    protected override JobHandle OnUpdate(JobHandle inputDependencies)
    {           
       var job = new RotationSpeedJob()
       {
           RotationType = GetArchetypeChunkComponentType<RotationQuaternion>(false),  // 구조체 필드 설정
           RotationSpeedType = GetArchetypeChunkComponentType<RotationSpeed>(true), // 구조체 필드 설정
           DeltaTime = Time.deltaTime // 구조체 필드 설정
       };

       return job.Schedule(m_Group, inputDependencies); //작업 예약
    }
    ```
    
  - 수동 반복(Manual iteration)
    NativeArray 에서 모든 청크를 명시적으로 요청하고 IJobParallelFor 같은 작업으로 처리 할 수도 있다.   
    EntityQuery 같은 모든 청크에 대한 단순 반복형 모델이 청크 관리에 적합하지 않은 경우 권장된다.
    
    ```csharp
    public class RotationSpeedSystem : JobComponentSystem
    {
       [BurstCompile]
       struct RotationSpeedJob : IJobParallelFor
       {
           [DeallocateOnJobCompletion] public NativeArray<ArchetypeChunk> Chunks;
           public ArchetypeChunkComponentType<RotationQuaternion> RotationType;
           [ReadOnly] public ArchetypeChunkComponentType<RotationSpeed> RotationSpeedType;
           public float DeltaTime;

           public void Execute(int chunkIndex)
           {
               var chunk = Chunks[chunkIndex];
               var chunkRotation = chunk.GetNativeArray(RotationType);
               var chunkSpeed = chunk.GetNativeArray(RotationSpeedType);
               var __instanceCount __= chunk.Count;

               for (int i = 0; i < instanceCount; i++)
               {
                   var rotation = chunkRotation[i];
                   var speed = chunkSpeed[i];
                   rotation.Value = math.mul(math.normalize(rotation.Value), quaternion.AxisAngle(math.up(), speed.RadiansPerSecond * DeltaTime));
                   chunkRotation[i] = rotation;
               }
           }
       }

       EntityQuery m_group;   

       protected override void OnCreate()
       {
           var query = new EntityQueryDesc
           {
               All = new ComponentType[]{ typeof(RotationQuaternion), ComponentType.ReadOnly<RotationSpeed>() }
           };

           m_group = GetEntityQuery(query);
       }

       protected override JobHandle OnUpdate(JobHandle inputDeps)
       {
           var rotationType = GetArchetypeChunkComponentType<RotationQuaternion>();
           var rotationSpeedType = GetArchetypeChunkComponentType<RotationSpeed>(true);
           var chunks = m_group.CreateArchetypeChunkArray(Allocator.__TempJob__);

           var rotationsSpeedJob = new RotationSpeedJob
           {
               Chunks = chunks,
               RotationType = rotationType,
               RotationSpeedType = rotationSpeedType,
               DeltaTime = Time.deltaTime
           };
           return rotationsSpeedJob.Schedule(chunks.Length,32,inputDeps);
       }
    }
    ```

---
## Burst Compiler

---
# 참고 자료
Unity DOTS 관련 문서
https://docs.unity3d.com/Packages/com.unity.entities@0.11/api/Unity.Entities.html
