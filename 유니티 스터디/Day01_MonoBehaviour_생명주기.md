# Day 01 — MonoBehaviour 생명주기 (Lifecycle)

> 한 줄 요약: 유니티가 스크립트의 메서드를 **정해진 순서와 타이밍**으로 자동 호출하는 규칙.
> 분류: Part 1 (유니티 코어) / 난이도: ★★☆☆☆ / 면접 빈출도: ★★★★★

## 1. 개념 (Why & What)

유니티에서 `MonoBehaviour`를 상속한 스크립트를 GameObject에 붙이면, 우리가 직접 호출하지 않아도 엔진이 특정 메서드들을 **생명주기(life cycle)**에 따라 알아서 불러줍니다. `Awake()`, `Start()`, `Update()` 같은 것들이죠.

왜 중요한가: "초기화 코드를 어디에 둘 것인가"가 이 순서에 달려 있습니다. 순서를 모르면 *"분명히 값을 넣었는데 null이다"* 같은 버그가 생깁니다. 면접에서 가장 기본이자 가장 자주 나오는 주제입니다.

## 2. 동작 원리 (호출 순서)

오브젝트 하나가 생성되어 파괴될 때까지의 주요 흐름:

```
[ 생성 / 초기화 ]
Awake()        → 오브젝트 인스턴스가 만들어질 때 1회. (비활성 상태여도 호출됨)
OnEnable()     → 오브젝트가 활성화될 때마다 호출 (활성/비활성 토글 시 반복)
Start()        → 첫 Update 직전에 1회. (활성 상태일 때만)

[ 매 프레임 반복 ]
FixedUpdate()  → 물리 타임스텝마다 (프레임과 별개, 고정 간격)
Update()       → 매 프레임 1회
LateUpdate()   → 모든 Update가 끝난 뒤 매 프레임 1회

[ 종료 ]
OnDisable()    → 비활성화될 때
OnDestroy()    → 파괴될 때 1회
```

**핵심 규칙 1 — Awake vs Start의 차이**
- `Awake`는 **자기 자신의 초기화** (변수 세팅, GetComponent 캐싱).
- `Start`는 **다른 오브젝트를 참조하는 초기화**. 모든 오브젝트의 `Awake`가 끝난 뒤에 `Start`가 돌기 때문에, "다른 애가 먼저 준비돼 있어야 하는" 코드는 Start에 둡니다.
- 즉, 모든 객체의 Awake → 모든 객체의 Start 순서가 보장됩니다.

**핵심 규칙 2 — Awake는 비활성이어도 호출, Start는 안 됨**
- `Awake`/`OnEnable`은 컴포넌트가 꺼져 있어도(비활성 GameObject) 활성화되는 순간 불립니다.
- 비활성 상태로 씬에 있으면 `Start`는 활성화될 때까지 미뤄집니다.

## 3. 코드 예제

```csharp
using UnityEngine;

public class Player : MonoBehaviour
{
    private Rigidbody _rb;

    void Awake()
    {
        // 자기 자신 초기화: 컴포넌트 캐싱은 여기서
        _rb = GetComponent<Rigidbody>();
    }

    void OnEnable()
    {
        // 활성화될 때마다: 이벤트 구독에 적합
        GameEvents.OnPause += HandlePause;
    }

    void Start()
    {
        // 다른 객체 참조: GameManager가 Awake에서 준비됐다고 가정
        transform.position = GameManager.Instance.SpawnPoint;
    }

    void Update()
    {
        // 입력 처리, 일반 로직 (프레임 의존 → deltaTime 사용)
        float move = Input.GetAxis("Horizontal") * Time.deltaTime;
    }

    void FixedUpdate()
    {
        // 물리 처리는 여기 (Rigidbody에 힘 가하기 등)
        _rb.AddForce(Vector3.forward);
    }

    void OnDisable()
    {
        // 구독 해제 (메모리 누수/중복 호출 방지)
        GameEvents.OnPause -= HandlePause;
    }

    private void HandlePause(bool paused) { /* ... */ }
}
```

## 4. 면접 Q&A

**Q1. Awake와 Start의 차이는?**
**A.** 둘 다 초기화용이지만 타이밍과 용도가 다릅니다. Awake는 오브젝트가 생성될 때 가장 먼저, 비활성 상태여도 호출되며 자기 자신의 초기화(컴포넌트 캐싱)에 씁니다. Start는 모든 객체의 Awake가 끝난 뒤 첫 Update 직전에 호출되어, 다른 객체를 참조하는 초기화에 적합합니다. 그래서 "준비 순서" 의존성이 있으면 참조하는 쪽을 Start에 둡니다.

**Q2. OnEnable/OnDisable은 언제 쓰나?**
**A.** 오브젝트가 활성/비활성될 때마다 반복 호출되므로 이벤트 구독/해제, 풀링된 오브젝트의 상태 리셋에 씁니다. 특히 오브젝트 풀링에서 꺼냈다 넣었다 할 때 OnEnable에서 초기화하는 패턴이 많습니다.

**Q3. 여러 오브젝트의 Awake 순서를 보장할 수 있나?**
**A.** 같은 객체 내 순서(Awake→Start)는 보장되지만, 서로 다른 객체 간 Awake 호출 순서는 기본적으로 비결정적입니다. 꼭 제어해야 하면 Script Execution Order 설정으로 특정 스크립트를 먼저 돌게 하거나, 의존성을 Start 단계로 미루는 방식으로 해결합니다.

## 5. 실수 / 함정 포인트

- **Awake에서 다른 객체 참조하기**: 그 객체의 Awake가 아직 안 돌았을 수 있어 null. → Start로 미루기.
- **OnEnable에서 구독하고 OnDisable에서 해제 안 함**: 비활성/파괴 후에도 이벤트가 살아남아 누수·중복 호출·NullReference 발생.
- **Update에 deltaTime 안 곱하기**: 프레임률에 따라 속도가 달라짐. 프레임 의존 값엔 항상 `Time.deltaTime`.
- **Start가 안 불린다고 당황**: 비활성 GameObject라 활성화 전까지 호출 보류된 것일 수 있음.

## 6. 한 줄 정리 (암기용)

> **Awake(나 자신) → OnEnable(켜질 때마다) → Start(남을 참조) → Update(매 프레임) → OnDisable/OnDestroy(끌 때/없앨 때).** 비활성이면 Awake는 불려도 Start는 미뤄진다.
