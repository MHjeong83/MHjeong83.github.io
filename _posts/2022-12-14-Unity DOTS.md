---
title: Unity DOTS 무작정 따라해보기!
categories: [Unity, DOTS]
---

# Unity ECS 실습

## 참고자료

- [https://docs.unity3d.com/Packages/com.unity.entities@1.0/manual/index.html](https://docs.unity3d.com/Packages/com.unity.entities@1.0/manual/index.html)
- [https://www.youtube.com/watch?v=H7zAORa3Ux0](https://www.youtube.com/watch?v=H7zAORa3Ux0)

# 준비물

- Unity 2022.2.0b16
- PackageManager 설치
    - com.unity.entities (1.0.0-exp.12)
    - [com.unity.entities.graphics](http://com.unity.entities.graphics) (1.00-exp.14)

- Project Settings 설정
    - Editor → Enter Play Mode Settings → Enter Play Mode Opertion, Disable Scene Backup 체크
    - Player → Other Settings → Configuration
    - Scripting Backend → IL2CPP
    - Api Compatibility Level → .Net FrameWork 설정
    - Scripting Define Symbol → ENABLE_TRANSFORM_V1 추가


# 서브씬 생성

- 선택된 씬에서 써브씬 생성함
- 써브신에서 Entity System 관련 코드들이 작동해야함
- Hierarchy - + - Create - New Sub Scene - 원하는 위치에 생성

# ECS 1.0

## IComponentData

```csharp
// Speed.cs
using Unity.Entities;

public struct Speed : IComponentData
{
	public float Value;
}
```

## Authoring & Baker

```csharp
// SpeedAuthoring.cs

using Unity.Entities;
using UnityEngine;

public class SpeedAuthoring : MonoBehaviour
{
    public float Value;
}

public class SpeedBaker : Baker<SpeedAuthoring>
{
    public override void Bake(SpeedAuthoring authoring)
    {
        AddComponent(new Speed
        {
            Value = authoring.Value
        });
    }
}
```

## 기본 사용법

<img width="1605" alt="img_001" src="https://user-images.githubusercontent.com/87407088/207586381-22584ea6-8a39-4e10-806b-5f6a37717d4a.png">


- 서브씬에 게임 오브젝트를 생성한 뒤 `SpeedAuthoring.cs` 를 컴포넌트에 추가한다
- 이렇게 보면 기존과 별다르지 않지만, `Inspector` 창 우측 상단에 자물쇠 아이콘 외쪽에 동그라미 버튼이 있을것이다. `클릭 → Runtime` 으로 변경하면 `Inspector` 창이 변경되는걸 볼 수 있습니다.

<img width="1605" alt="img_002" src="https://user-images.githubusercontent.com/87407088/207586453-1fa7f867-4d75-4a12-97ef-fe2bfeb89676.png">


## SystemBase

- Class - Managed
- Simpler, runs on main thread, canot use Burst
- 아래에 코드를 작성한 뒤 실행을 하면 움직이는걸 볼 수 있습니다.

```csharp
// MovingSystemBase.cs
// partial 필수!!!

using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

public partial class MovingSystemBase : SystemBase
{
    protected override void OnUpdate()
    {
        foreach (TransformAspect transformAspect in SystemAPI.Query<TransformAspect>())
        {
            transformAspect.Position += new float3(SystemAPI.Time.DeltaTime, 0, 0);
        }
    }
}
```

- Windows - Entities - Systems - 확인 가능

<img width="728" alt="img_003" src="https://user-images.githubusercontent.com/87407088/207586511-eaa6af89-1d3c-4f01-a3c4-8c99a63b06d9.png">

## ISystem

- Struct - Unmanaged
- Can use Burst, extremely fast but slightly more complex

## 이동 목적지 설정하기

```csharp
// TargetPosition.cs

using Unity.Entities;
using Unity.Mathematics;

public struct TargetPosition : IComponentData
{
    public float3 Value;
}
```

```csharp
// TargetPositionAuthoring.cs

using Unity.Entities;
using Unity.Mathematics;
using UnityEngine;

public class TargetPositionAuthoring : MonoBehaviour
{
    public float3 Value;
}

public class TargetPositionBaker : Baker<TargetPositionAuthoring>
{
    public override void Bake(TargetPositionAuthoring authoring)
    {
        AddComponent(new TargetPosition
        {
            Value = authoring.Value,
        });
    }
}
```

```csharp
// MovingSystemBase.cs 수정
// RefRO -> ReadOnly
// RefRW -> Read/Write

using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

public partial class MovingSystemBase : SystemBase
{
    protected override void OnUpdate()
    {
        foreach ((TransformAspect transformAspect, RefRO<Speed> speed, RefRW<TargetPosition> targetPosition) in
                 SystemAPI.Query<TransformAspect, RefRO<Speed>, RefRW<TargetPosition>>())
        {
            // Calculate dir
            float3 direction = math.normalize(targetPosition.ValueRW.Value - transformAspect.Position);
            // Move
            transformAspect.Position += direction * SystemAPI.Time.DeltaTime * speed.ValueRO.Value;
        }
    }
}
```

<img width="1605" alt="img_004" src="https://user-images.githubusercontent.com/87407088/207586581-3271ff7b-1194-48fa-9546-880cef221925.png">

- 해당 컴포넌트를 추가 한뒤 값을 설정해주고 실행을 해보면 설정한 스피드로 목적지까지 가는걸 확인합니다.

## IAsect을 활용하여 랜덤하게 이동해보기

- [https://docs.unity3d.com/Packages/com.unity.entities@1.0/manual/aspects-create.html](https://docs.unity3d.com/Packages/com.unity.entities@1.0/manual/aspects-create.html)

```csharp
// MovingSystemBase.cs 수정

using Unity.Entities;

public partial class MovingSystemBase : SystemBase
{
    protected override void OnUpdate()
    {
        RefRW<RandomComponent> randomComponent = SystemAPI.GetSingletonRW<RandomComponent>();

        foreach (MoveToPositionAspect moveToPositionAspect in SystemAPI.Query<MoveToPositionAspect>())
        {
            moveToPositionAspect.Move(SystemAPI.Time.DeltaTime, randomComponent);
        }
    }
}
```

```csharp
// MovoToPositionAspect.cs

using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

public readonly partial struct MoveToPositionAspect : IAspect
{
    private readonly Entity _entity;

    private readonly TransformAspect _transformAspect;
    private readonly RefRO<Speed> _speed;
    private readonly RefRW<TargetPosition> _targetPosition;

    public void Move(float deltaTime, RefRW<RandomComponent> randomComponent)
    {
        // Calculate dir
        float3 direction = math.normalize(_targetPosition.ValueRW.Value - _transformAspect.Position);
        // Move
        _transformAspect.Position += direction * deltaTime * _speed.ValueRO.Value;

        float reachedTargetDistance = .5f;
        if (math.distance(_transformAspect.Position, _targetPosition.ValueRW.Value) < reachedTargetDistance)
        {
            // Generate new random target position
            _targetPosition.ValueRW.Value = GetRandomPosition(randomComponent);
        }
    }

    private float3 GetRandomPosition(RefRW<RandomComponent> randomComponent)
    {
        return new float3(
            randomComponent.ValueRW.Random.NextFloat(0f, 15f),
            0,
            randomComponent.ValueRW.Random.NextFloat(0f, 15f)
        );
    }
}
```

```csharp
// RancomComponent.cs

using Unity.Entities;

public struct RandomComponent : IComponentData
{
    public Unity.Mathematics.Random Random;
}
```

```csharp
// RandomAuthoring.cs

using Unity.Entities;
using UnityEngine;

public class RandomAuthoring : MonoBehaviour
{
}

public class RandomBaker : Baker<RandomAuthoring>
{
    public override void Bake(RandomAuthoring authoring)
    {
        AddComponent(new RandomComponent
        {
            Random = new Unity.Mathematics.Random(1)
        });
    }
}
```

<img width="1605" alt="img_005" src="https://user-images.githubusercontent.com/87407088/207586638-7d57f3e0-128b-4201-bb0a-1eb3a4489474.png">


- 위처럼 구성한 뒤 실행 해보면 랜덤하게 이동하는걸 확인 할 수 있다.

## ISystem, JobSystem을 활용해보자!(+BurstCompile)

```csharp
// MovingSystem.cs 수정

using Unity.Burst;
using Unity.Collections.LowLevel.Unsafe;
using Unity.Entities;
using Unity.Jobs;

[BurstCompile]
public partial struct MovingISystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {

    }

    [BurstCompile]
    public void OnDestroy(ref SystemState state)
    {

    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        RefRW<RandomComponent> randomComponent = SystemAPI.GetSingletonRW<RandomComponent>();

        float deltaTime = SystemAPI.Time.DeltaTime;

        JobHandle jobHandle = new MoveJob
        {
            DeltaTime = deltaTime
        }.ScheduleParallel(state.Dependency);

        jobHandle.Complete();

        new TestReachedTargetPositionJob
        {
            RandomComponent = randomComponent
        }.Run();
    }
}

[BurstCompile]
public partial struct MoveJob : IJobEntity
{
    public float DeltaTime;

    public void Execute(MoveToPositionAspect moveToPositionAspect)
    {
        moveToPositionAspect.Move(DeltaTime);
    }
}

[BurstCompile]
public partial struct TestReachedTargetPositionJob : IJobEntity
{
    [NativeDisableUnsafePtrRestriction] public RefRW<RandomComponent> RandomComponent;

    public void Execute(MoveToPositionAspect moveToPositionAspect)
    {
        moveToPositionAspect.TestReachedTargetPosition(RandomComponent);
    }
}
```

```csharp
// MoveToPositionAspect.cs 수정

using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

public readonly partial struct MoveToPositionAspect : IAspect
{
    private readonly Entity _entity;

    private readonly TransformAspect _transformAspect;
    private readonly RefRO<Speed> _speed;
    private readonly RefRW<TargetPosition> _targetPosition;

    public void Move(float deltaTime)
    {
        // Calculate dir
        float3 direction = math.normalize(_targetPosition.ValueRW.Value - _transformAspect.Position);
        // Move
        _transformAspect.Position += direction * deltaTime * _speed.ValueRO.Value;
    }

    public void TestReachedTargetPosition(RefRW<RandomComponent> randomComponent)
    {
        float reachedTargetDistance = .5f;
        if (math.distance(_transformAspect.Position, _targetPosition.ValueRW.Value) < reachedTargetDistance)
        {
            // Generate new random target position
            _targetPosition.ValueRW.Value = GetRandomPosition(randomComponent);
        }
    }

    private float3 GetRandomPosition(RefRW<RandomComponent> randomComponent)
    {
        return new float3(
            randomComponent.ValueRW.Random.NextFloat(0f, 15f),
            0,
            randomComponent.ValueRW.Random.NextFloat(0f, 15f)
        );
    }
}
```

```csharp
// MovingSystemBase.cs 수정

using Unity.Entities;

public partial class MovingSystemBase : SystemBase
{
    protected override void OnUpdate()
    {
        RefRW<RandomComponent> randomComponent = SystemAPI.GetSingletonRW<RandomComponent>();

        foreach (MoveToPositionAspect moveToPositionAspect in SystemAPI.Query<MoveToPositionAspect>())
        {
            moveToPositionAspect.Move(SystemAPI.Time.DeltaTime);
            moveToPositionAspect.TestReachedTargetPosition(randomComponent);
        }
    }
}
```

- 프로파일링 해봅시다~!

<img width="1760" alt="img_006" src="https://user-images.githubusercontent.com/87407088/207586683-a99212b2-759a-4e64-bc26-76662b61d170.png">

<img width="1760" alt="img_007" src="https://user-images.githubusercontent.com/87407088/207586719-3817396a-4d8d-4782-bc99-602979be4523.png">

-
- 확인완료!

## 스트레스 테스트를 진행해보자!

```csharp
// PlayerSpawnerComponent.cs

using Unity.Entities;

public struct PlayerSpawnerComponent : IComponentData
{
    public Entity PlayerPrefab;
}
```

```csharp
// PlayerSpawnerAuthoring.cs

using Unity.Entities;
using UnityEngine;

public class PlayerSpawnerAuthoring : MonoBehaviour
{
    public GameObject PlayerPrefab;
}

public class PlayerSpawnerBaker : Baker<PlayerSpawnerAuthoring>
{
    public override void Bake(PlayerSpawnerAuthoring authoring)
    {
        AddComponent(new PlayerSpawnerComponent
        {
            PlayerPrefab = GetEntity(authoring.PlayerPrefab),
        });
    }
}
```

```csharp
// PlayerSpawnerSystem.cs

using Unity.Entities;

public partial class PlayerSpawnerSystem : SystemBase
{
    protected override void OnUpdate()
    {
        EntityQuery playerEntityQuery = EntityManager.CreateEntityQuery(typeof(PlayerTag));

        PlayerSpawnerComponent playerSpawnerComponent = SystemAPI.GetSingleton<PlayerSpawnerComponent>();
        RefRW<RandomComponent> randomComponent = SystemAPI.GetSingletonRW<RandomComponent>();

        EntityCommandBuffer entityCommandBuffer = SystemAPI.GetSingleton<BeginSimulationEntityCommandBufferSystem.Singleton>()
            .CreateCommandBuffer(World.Unmanaged);

        int spawnAmount = 30000;
        if (playerEntityQuery.CalculateEntityCount() < spawnAmount)
        {
            // EntityManager.Instantiate(playerSpawnerComponent.PlayerPrefab);
            Entity spawnEntity = entityCommandBuffer.Instantiate(playerSpawnerComponent.PlayerPrefab);
            entityCommandBuffer.SetComponent(spawnEntity, new Speed
            {
                Value = randomComponent.ValueRW.Random.NextFloat(1f, 5f)
            });
        }
    }
}
```

```csharp
// PlayerTag.cs

using Unity.Entities;

public struct PlayerTag : IComponentData
{
}
```

```csharp
// PlayerTagAuthoring.cs

using Unity.Entities;
using UnityEngine;

public class PlayerTagAuthoring : MonoBehaviour
{

}

public class PlayerTagBaker : Baker<PlayerTagAuthoring>
{
    public override void Bake(PlayerTagAuthoring authoring)
    {
        AddComponent(new PlayerTag());
    }
}
```

<img width="2168" alt="img_008" src="https://user-images.githubusercontent.com/87407088/207586758-4ac8f69f-ba54-49d3-b5d4-aea84819611f.png">


<img width="2168" alt="img_009" src="https://user-images.githubusercontent.com/87407088/207586764-6b6a64b6-96c2-494b-b363-fb8868407478.png">


음,,, 뭔가 기대 이상은 아닌것같은데,,,,

무빙에 관련하는 부분이 2곳이 있는것 같은데  `Moving System Base` 를 꺼보자

<img width="2168" alt="img_010" src="https://user-images.githubusercontent.com/87407088/207586815-d964044d-540b-4f66-b997-66fd370d3205.png">


오,,,? 이제서야 뭔가 원하는 프레임이 나온듯하다

간단한 예제를 사용하며 따라해보았으니

기능들에 대해서 알아가는 시간을 가져야겠습니다.
