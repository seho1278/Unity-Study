# Day 03 — 게임 루프와 프레임, Time.deltaTime / timeScale

> 한 줄 요약: 게임은 매 프레임 "입력→갱신→렌더"를 반복하며, 프레임 간격(`deltaTime`)을 곱해 **속도를 프레임률과 무관하게** 만든다.
> 분류: Part 1 (유니티 코어) / 난이도: ★★☆☆☆ / 면접 빈출도: ★★★★☆

## 1. 개념 (Why & What)

게임 화면은 정지 그림이 아니라 1초에 수십 장의 정지 화면(프레임)을 빠르게 갈아끼우는 것입니다. 그 한 장을 그리기 위해 매번 도는 반복 구조가 **게임 루프(game loop)**입니다. 유니티는 이 루프를 엔진이 돌려주고, 우리는 그 안의 `Update` 같은 콜백에 코드를 끼워 넣는 것이죠.

핵심 문제: 프레임률(FPS)은 기기와 부하에 따라 변합니다. 같은 코드라도 60fps 기기에선 부드럽고 30fps 기기에선 느리게 보일 수 있습니다. 이 **프레임률 의존성**을 없애는 도구가 `Time.deltaTime`입니다. 면접에서 "왜 deltaTime을 곱하나요?"는 기본기 검증용 단골 질문입니다.

## 2. 동작 원리

**게임 루프 한 바퀴 = 1 프레임**
개념적으로 엔진은 매 프레임 이런 순서를 돕니다:

```
한 프레임:
1) 입력 수집 (Input)
2) 물리 업데이트 (FixedUpdate들 — 누적 시간만큼 0~N회)
3) 로직 업데이트 (Update → LateUpdate)
4) 렌더링 (화면에 그리기)
5) 다음 프레임으로
```

**deltaTime — "이번 프레임이 그려지는 데 걸린 시간(초)"**
- 60fps면 `deltaTime ≈ 0.0166`, 30fps면 `≈ 0.0333`.
- 프레임당 변화량에 deltaTime을 곱하면 "초당 변화량"으로 환산됩니다.
- 예: `speed = 5`일 때 `position += speed * Time.deltaTime`을 매 프레임 하면, 프레임률이 몇이든 **1초에 정확히 5만큼** 이동합니다. (60fps면 작은 양 많이, 30fps면 큰 양 적게 → 합은 같음)

**프레임률에 곱하지 않으면?**
`position += speed` (deltaTime 없이) 하면 프레임마다 5씩 → 60fps에선 1초에 300, 30fps에선 150. 기기마다 속도가 달라지는 버그.

**fixedDeltaTime — 물리의 고정 간격**
FixedUpdate는 고정 주기(기본 0.02초)로 돌고, 그 안에서의 시간 변수는 `Time.fixedDeltaTime`입니다. Day 02 참고.

**timeScale — 시간의 흐름 배속**
- `Time.timeScale = 1` 정상, `0` 완전 정지(일시정지), `0.5` 슬로우모션, `2` 2배속.
- `Time.deltaTime`과 FixedUpdate 주기가 timeScale의 영향을 받습니다.
- 주의: **UI 애니메이션이나 일시정지 메뉴**는 timeScale=0이어도 움직여야 하므로 `Time.unscaledDeltaTime`을 써야 합니다.

| 변수 | 의미 | timeScale 영향 |
|------|------|----------------|
| `Time.deltaTime` | 직전 프레임 소요 시간 | 받음 |
| `Time.fixedDeltaTime` | 물리 고정 스텝 | 받음 |
| `Time.unscaledDeltaTime` | timeScale 무시한 실제 시간 | 안 받음 |
| `Time.time` | 게임 시작 후 누적 시간 | 받음 |

## 3. 코드 예제

```csharp
using UnityEngine;

public class TimeExamples : MonoBehaviour
{
    [SerializeField] private float speed = 5f;     // 초당 5유닛
    [SerializeField] private float rotSpeed = 90f; // 초당 90도

    void Update()
    {
        // 프레임률과 무관하게 일정한 이동/회전
        transform.position += Vector3.forward * speed * Time.deltaTime;
        transform.Rotate(0, rotSpeed * Time.deltaTime, 0);
    }

    // 일시정지 토글 (timeScale 0 ↔ 1)
    public void TogglePause()
    {
        Time.timeScale = Time.timeScale == 0 ? 1 : 0;
    }
}

public class PauseMenuAnimator : MonoBehaviour
{
    void Update()
    {
        // 일시정지(timeScale=0) 중에도 메뉴는 움직여야 하므로 unscaled 사용
        float t = Mathf.PingPong(Time.unscaledTime, 1f);
        transform.localScale = Vector3.one * (0.9f + 0.1f * t);
    }
}
```

## 4. 면접 Q&A

**Q1. 왜 이동/회전에 Time.deltaTime을 곱하나요?**
**A.** 프레임률이 기기마다 다르기 때문입니다. deltaTime은 직전 프레임이 그려진 시간(초)이라, 프레임당 변화량에 곱하면 초당 변화량으로 환산됩니다. 그 결과 30fps든 120fps든 1초에 이동하는 총량이 같아져, 기기 성능과 무관하게 일정한 속도를 보장합니다. 안 곱하면 빠른 기기에서 더 빨리 움직이는 버그가 생깁니다.

**Q2. 게임을 일시정지하려면 어떻게 하나요? 그 방식의 단점은?**
**A.** 가장 간단한 방법은 `Time.timeScale = 0`입니다. deltaTime이 0이 되고 FixedUpdate도 멈춰 물리·애니메이션이 정지합니다. 단점은 timeScale에 의존하지 않아야 할 것들(일시정지 메뉴 UI 애니메이션, 입력 처리)까지 멈춘다는 점입니다. 그런 부분은 `Time.unscaledDeltaTime`을 쓰거나, 코루틴의 `WaitForSecondsRealtime`을 사용해 따로 처리해야 합니다.

**Q3. deltaTime을 곱했는데도 빠른 물체가 벽을 뚫는 이유는?**
**A.** 터널링(tunneling) 문제입니다. 한 물리 스텝 동안 이동 거리가 충돌체 두께보다 크면 충돌 검사 사이를 통과해버립니다. 이건 deltaTime 보정과 별개 문제로, Rigidbody의 Collision Detection을 Continuous로 바꾸거나, 물리 스텝을 더 잘게 하거나, 레이캐스트로 직접 검사해 해결합니다.

## 5. 실수 / 함정 포인트

- **deltaTime 누락**: 프레임률 의존 속도. 가장 흔한 실수.
- **deltaTime을 한 번 더 곱함**: 가속도처럼 `velocity += accel * dt` 후 `pos += velocity * dt`는 맞지만, 단순 이동에 dt를 두 번 곱하면 비정상적으로 느려짐.
- **timeScale=0 상태에서 코루틴 `WaitForSeconds`가 멈춤**: timeScale 영향을 받기 때문. 정지 중에도 돌아야 하면 `WaitForSecondsRealtime`.
- **입력을 timeScale로 막으려 함**: 입력 자체는 멈추지 않음. 일시정지 로직에서 별도 처리 필요.
- **deltaTime을 물리(FixedUpdate)에서 쓸지 헷갈림**: 의미상 `fixedDeltaTime`이 맞음.

## 6. 한 줄 정리 (암기용)

> **프레임률은 들쭉날쭉 → 변화량에 `Time.deltaTime`(초) 곱해 속도를 통일.** 일시정지는 `timeScale=0`, 단 정지 중에도 돌 UI/입력은 `unscaledDeltaTime`/`WaitForSecondsRealtime`.
