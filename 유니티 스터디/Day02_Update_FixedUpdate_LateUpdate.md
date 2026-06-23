# Day 02 — Update vs FixedUpdate vs LateUpdate

> 한 줄 요약: 매 프레임 도는 세 가지 갱신 루프 — **타이밍과 호출 주기**가 다르므로 용도가 다르다.
> 분류: Part 1 (유니티 코어) / 난이도: ★★☆☆☆ / 면접 빈출도: ★★★★★

## 1. 개념 (Why & What)

유니티에서 "매 프레임 실행되는 코드"를 넣을 곳이 세 군데 있습니다. 셋 다 반복 호출되지만 **언제, 얼마나 자주** 불리는지가 다릅니다. 이걸 모르고 물리 코드를 Update에 넣으면 프레임률에 따라 결과가 흔들리고, 카메라 추적을 Update에 넣으면 화면이 떨립니다. 면접에서 거의 100% 나오는 단골 비교 문제입니다.

| 구분 | 호출 주기 | 시간 변수 | 주 용도 |
|------|-----------|-----------|---------|
| `Update` | 매 프레임 1회 (가변) | `Time.deltaTime` | 입력, 일반 게임 로직 |
| `FixedUpdate` | 고정 간격 (기본 0.02초) | `Time.fixedDeltaTime` | 물리 연산 (Rigidbody) |
| `LateUpdate` | 매 프레임, 모든 Update 후 | `Time.deltaTime` | 카메라 추적, 후처리 |

## 2. 동작 원리

**Update — 프레임에 종속 (가변 주기)**
프레임마다 정확히 1회 호출됩니다. 그런데 프레임률은 기기/부하에 따라 변합니다 (60fps면 약 16.6ms 간격, 30fps면 33ms 간격). 그래서 Update에서 "이동량"을 계산할 땐 반드시 `Time.deltaTime`을 곱해 **프레임당 변화량 → 초당 변화량**으로 보정해야 일정한 속도가 나옵니다.

**FixedUpdate — 물리에 종속 (고정 주기)**
물리 엔진은 일정한 시간 간격으로 계산해야 안정적입니다(적분 오차 누적 방지). 그래서 FixedUpdate는 프레임과 무관하게 **고정된 물리 타임스텝**(Project Settings의 Fixed Timestep, 기본 0.02초 = 50회/초)마다 호출됩니다.
- 프레임이 느리면(예: 25fps) 한 프레임에 FixedUpdate가 **여러 번** 불릴 수 있고,
- 프레임이 빠르면(예: 200fps) **한 번도 안 불리는** 프레임도 생깁니다.
- Rigidbody에 힘을 가하거나 velocity를 다루는 코드는 여기에 둬야 합니다.

**LateUpdate — Update 다음 보장**
그 프레임의 **모든 객체의 Update가 끝난 뒤** 호출됩니다. 대표 용도는 카메라가 플레이어를 따라가는 코드. 플레이어가 Update에서 움직이는데 카메라도 Update에서 따라가면 호출 순서에 따라 한 프레임 늦은(떨리는) 위치를 잡을 수 있습니다. LateUpdate에 두면 플레이어 이동이 모두 끝난 "최종 위치"를 보고 따라가므로 떨림이 없습니다.

## 3. 코드 예제

```csharp
using UnityEngine;

public class Movement : MonoBehaviour
{
    [SerializeField] private float speed = 5f;
    private Rigidbody _rb;
    private Vector3 _input;

    void Awake() => _rb = GetComponent<Rigidbody>();

    void Update()
    {
        // 입력은 프레임마다 읽는다 (FixedUpdate에서 GetKeyDown 읽으면 놓칠 수 있음)
        _input = new Vector3(Input.GetAxis("Horizontal"), 0, Input.GetAxis("Vertical"));
    }

    void FixedUpdate()
    {
        // 물리 이동은 고정 주기에서. deltaTime이 아니라 fixedDeltaTime
        _rb.MovePosition(_rb.position + _input * speed * Time.fixedDeltaTime);
    }
}

public class CameraFollow : MonoBehaviour
{
    [SerializeField] private Transform target;
    private Vector3 _offset = new Vector3(0, 5, -10);

    // 플레이어 이동이 끝난 뒤 따라가야 떨리지 않는다
    void LateUpdate()
    {
        transform.position = target.position + _offset;
    }
}
```

## 4. 면접 Q&A

**Q1. Update와 FixedUpdate의 차이는?**
**A.** Update는 매 프레임 1회 호출되며 프레임률에 따라 호출 간격이 가변적입니다. FixedUpdate는 고정된 물리 타임스텝(기본 0.02초)마다 호출되어 프레임과 무관하게 일정 주기로 돕니다. 그래서 일반 로직·입력은 Update에, Rigidbody 같은 물리 연산은 FixedUpdate에 둡니다. 물리를 일정 간격으로 계산해야 시뮬레이션이 안정적이기 때문입니다.

**Q2. 입력 처리(GetKeyDown)는 어디서 해야 하나?**
**A.** Update입니다. GetKeyDown/GetKeyUp 같은 "그 프레임에만 true"인 입력은 매 프레임 검사하는 Update에서 읽어야 합니다. FixedUpdate는 프레임당 0번 불릴 수도 있어서 입력을 놓칠 수 있습니다. 보통 Update에서 입력을 받아 변수에 저장하고, FixedUpdate에서 그 값으로 물리를 처리합니다.

**Q3. 카메라 추적은 왜 LateUpdate에 두나?**
**A.** LateUpdate는 모든 Update가 끝난 뒤 호출되므로, 플레이어가 Update에서 이동을 마친 최종 위치를 보고 카메라가 따라갑니다. Update에 두면 플레이어와 카메라의 호출 순서에 따라 한 프레임 뒤처진 위치를 추적해 화면이 떨릴 수 있습니다.

**Q4. 프레임이 한 번 도는 동안 FixedUpdate가 몇 번 불리나?**
**A.** 정해져 있지 않습니다. 프레임률과 Fixed Timestep에 따라 0번, 1번, 또는 여러 번 불릴 수 있습니다. 누적된 시간이 Fixed Timestep을 넘을 때마다 호출되며, 프레임이 느리면 한 프레임에 여러 번 몰아서 호출됩니다.

## 5. 실수 / 함정 포인트

- **물리 코드를 Update에 넣기**: 프레임률에 따라 힘/속도가 달라지고 충돌이 불안정. → FixedUpdate.
- **FixedUpdate에서 deltaTime 사용**: `Time.deltaTime`은 FixedUpdate 안에서는 자동으로 `fixedDeltaTime` 값을 반환하긴 하지만, 의미를 명확히 하려면 `Time.fixedDeltaTime`을 쓰는 게 안전.
- **FixedUpdate에서 GetKeyDown 읽기**: 입력 누락 발생.
- **카메라/본 추적을 Update에**: 떨림(jitter) 발생. → LateUpdate.
- **무거운 연산을 Update에 매 프레임**: 성능 저하. 꼭 매 프레임 필요한지 점검 (이벤트 기반/코루틴 분산).

## 6. 한 줄 정리 (암기용)

> **입력·로직은 Update(deltaTime), 물리는 FixedUpdate(고정 주기), 카메라 추적은 LateUpdate(Update 다 끝난 뒤).** FixedUpdate는 한 프레임에 0~N번 불릴 수 있다.
