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

#### IComponentData
- 범용 컴포넌트를 구현하기 위한 인터페이스
```csharp
[GenerateAuthoringComponent]
public struct ComponentData : IComponentData
{
    public float fComponentValue;
}
```
- **GenerateAuthoringComponent :** ECS 컴포넌트는 MonoBehaviour를 상속받는 존재가 아니기에 인스펙터에 노출 시킬수가 없다. 그래서 사용하는게 해당 속성인데, 구조체에 부여하면 엔티티의 인스펙터에 노출시킬 수 있게 되어 초기 값을 지정할 수 있게 된다.
- 

####
---
## Job System
