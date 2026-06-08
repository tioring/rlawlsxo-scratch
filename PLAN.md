# 디제이 입력 레이어 리팩토링 플랜 (v2 — 리뷰 반영)

> 대상: 단일 `index.html` (+ `soundtouch.js`). Web Audio API 듀얼덱 턴테이블리즘.
> 아키텍처: **B + worklet NaN 방어가드** (당초 "C 풀 추출"에서 리뷰로 정제 — 13절 참조).
> 근거: 다중관점 조사 4건 + 적대적 리뷰 2건(구현 실패지점 / 빠진 엣지케이스).

---

## 1. 한 줄 정의
흩어진 크로스페이더·스크래치 입력 핸들러를 **입력 레이어(`CrossfaderController` + `ScratchInput`)로 추출**하면서 ①크로스페이더 래치 컷 ②마우스 Pointer Lock 정밀 스크래치를 얹고, 클릭리스 게인 전이(`setParam`)와 velocity NaN 가드를 회수한다. **존재이유 = 왼손 컷 + 오른손 정밀 스크래치 양손 동시 연주.**

## 2. 목표 / 비목표
**Goals**: Shift+Q/W/E 토글 래치(Q/W 펄스·E 긴컷) · Pointer Lock 비선형 정밀 스크래치(Vinyl과 공존) · 스크래치 중복 정리 · 클릭리스 전이 · velocity NaN 방어 · **양손 동시 보장.**
**Non-goals**: SAB 채널(②) · worklet 프로토콜 대수술(NaN가드+선택적 smoothing 인자만) · 비트그리드 cache bake(④) · 죽은코드 제거(⑥⑦) · 자동 비트동기(폐기 전례).

## 3. 아키텍처

### 3.1 모듈 경계
```
keydown(:2117)/keyup/ESC(:1971) ──► CrossfaderController.onKey(code, shift, type)
                                        states: normal | latched
pointerdown/move/up, pointerlockchange ─► ScratchInput[deck]
                                        source: idle | platter | zoom | precision
공용 유틸:
  setParam(param, value, {mode:'immediate'|'cut'|'smooth'})   // 게인 전이 단일 창구
  sanitizeVel(v) = isFinite(v) ? clamp(v,-MAX,MAX) : null      // NaN/Inf 차단
  scratchCurve(normDelta, dt) = 비선형(γ), dt floor             // movement→vel
  anchorValueFromGeometry(deckId) = zoom중심 → crossfader 레일 값(DOM-relative)
  endScratch(deck) = idempotent, scratch-end 1회               // 모든 종료경로 공용
```
**입력 라우팅**: 새 리스너 금지. 기존 단일 keydown(:2117)·ESC(:1971)에서 컨트롤러 메서드 호출. Pointer Lock의 `movementX/Y`만 **document 레벨 별도 핸들러**(기존 window onMove와 분리).

### 3.2 상태머신 (리뷰 반영)

**CrossfaderController** — `crossfade` 값의 **단일 권위**. 모드별 디스패치:
```
디스패치 규칙(우선순위): onKey(code,shift,type)
  if shift && code∈{Q,W,E} && type==down  → 래치 토글(같은 키 재탭=해제)   [Shift keyup 무시]
  else if mode==normal && code∈CF_POINTS   → 기존 cfKeyStack(:2123) 보존
  else if mode==latched:
      Q,W down(!repeat) → 펄스: setParam('cut', 반대편) + 타이머(PULSE_MS) → 앵커 복귀
      E   down(!repeat) → setParam('cut', 반대편) 유지
      E   up            → 앵커 복귀                                  [holdable 긴컷]
rAF 이징(:2177): mode==normal 일 때만 cfTarget 추종. **latched 동안 추종 분기 스킵**
                 (컨트롤러가 crossfade 소유). normal 복귀 시 cfTarget=crossfade 동기화.
```

**ScratchInput (덱별)** — source별 mapping/state는 **분리 유지**(평탄화 금지), 공용은 emit/sanitize/endScratch/visualAngle 계약만:
```
idle ─pointerdown(platter, vinyl)──────────► platter   (angular, deck.dragging, window onMove)
idle ─pointerdown(zoom)─────────────────────► zoom      (vertical-delta, 클로저 zDrag, 탭seek)
idle ─pointerdown(platter, precision)+lock획득► precision (locked-relative, document onMove, movementY)
종료(idle 복귀)·endScratch 트리거:
   platter/zoom: pointerup/cancel
   precision:    **pointerlockchange(해제) / 모드토글 / Esc 만**  ← pointerup은 종료 아님(락 유지)
   공통 추가:    blur / visibilitychange
visualAngle 계약: platter는 `visualAngle += totalDAngle`(직접) — emit이 onVisualDelta로 반환.
                 precision은 curVel 기반 회전(별도 경로, deck.dragging 미사용).
```

### 3.3 게인/메시지 흐름 (리뷰 반영)
- **`applyCrossfade`(:1231)는 `gain.value=` 즉시대입 유지** — rAF 이징(60Hz)이 이미 스무딩이므로 AudioParam 램프 불필요·유해(매 프레임 ramp 재생성=zipper+부하). **유일한 게인 라이터로 통일**: `ensureNode`(:1629)·커브전환(:2067)도 반드시 `applyCrossfade` 경유(현재 :1629 우회 → 교체).
- **`setParam('cut')`(linearRamp)은 래치 컷 이벤트 1회만**(Phase4). **`setParam('smooth')`(setTargetAtTime)은 연사 input 디지퍼용**: 크로스페이더 슬라이더 input(:2055)·EQ/볼륨 input(:1746-1749)·커브버튼(:2067). (이게 실제 zipper 발생처 — applyCrossfade가 아님.)
- scratch 메시지(`scratch-start/velocity/end`) 재사용. **worklet+fallback에 NaN 방어가드 추가**(수신 `if(!isFinite(value))return;` + position 갱신 직후 클램프, :727/:982/:854 양쪽 동기). **per-mode smoothing 낮추기는 폐기**(0.997은 chalk-noise 억제 — 낮추면 zipper 악화). smoothing은 유지, 반응성은 coalesced 샘플링으로.

## 4. 핵심 기술 결정 (API 검증)
| 항목 | 결정 |
|---|---|
| 컷 게인(래치 이벤트) | `cancelScheduledValues(now)`→`setValueAtTime(param.value, now+LA)`→`linearRampToValueAtTime(target, now+LA+CUT_MS)`. setTargetAtTime은 0 미도달이라 컷 부적합 |
| 연사 input 디지퍼 | `setTargetAtTime(v, now+LA, ~0.008)` — 누적 취소 없이 타깃만 갱신(폭주 X) |
| 스케줄 시각 | 항상 `currentTime+LA`(≈0.008s). FF currentTime 2ms 라운딩+스레드지연 |
| cancelAndHoldAtTime | 폴백(`cancelScheduledValues`+`setValueAtTime(현재값)`) — Limited availability |
| Pointer Lock 진입 | `requestPointerLock({unadjustedMovement:true})`→`NotSupportedError`면 옵션없이 재요청(raw=Chromium+Win/macOS 한정) |
| 락 상태 판정 | `pointerlockchange`/`error`로만(Promise 비표준, Esc keydown 미보장) |
| movementY | `dy/devicePixelRatio` 정규화→`scratchCurve`→vel. dt floor(≥1ms). **modifier 키 무시**(Shift는 CF 라우팅만) |
| AudioContext | `{latencyHint:'interactive'}` + `baseLatency` 로그(백로그①) |
| 키 충돌 | KeyE는 CF_POINTS 점유 → Shift분기. 래치=**탭 토글**(Shift+키 down 1회, Shift keyup 무시) |

## 5. 상태 구조
```js
const CUT_MS=0.004, PARAM_LA=0.008, SCRATCH_VEL_MAX=40, SCRATCH_GAMMA=1.6;
const PULSE_MS=90;     // Q/W 펄스 — declick fade(5ms)의 ≥2배 보장(아니면 클릭, 검증 F3)
const CF_CUTLAG=0.0;   // 끝단 데드존(튜너블)
// CrossfaderController: mode, anchor, anchorSide, latchKey, pulseTimer
// deck: scratchSource, scratchMode('vinyl'|'precision'), lockActive, precisionSens
```

## 6. 핵심 플로우
1. **양손 동시(존재이유)**: 오른손 Precision 락 스크래치(커서숨김, 버튼 떼고 이동) **진행 중** 왼손 `Shift+Q` 탭=래치 → Q/W 연타 펄스컷·E 긴컷. 키는 window로 흐르고 락은 마우스만 잡으므로 **동시 성립**(검증 F1 필수 PASS).
2. **회귀 불변**: normal 크로스페이더(크랩·단타우선·마우스우선) · Vinyl 각속도(드래그 중 판 회전) · zoom 세로스크럽·탭seek·관성.

## 7. 엣지케이스 정책 (리뷰 도출 — 구현 전 확정)
| | 상황 | 정책 |
|---|---|---|
| A1 | 래치 ON + 새 트랙 로드 | 래치 유지(앵커=레일값, 덱무관). 로드 중 펄스 타이머 안전 클리어 |
| A2 | Precision + 트랙 미로드 플래터 클릭 | 락 요청은 `deck.buffer` 가드 **통과 후에만**(빈 덱 클릭 시 커서 안 숨김) |
| A3 | 래치 중 마우스로 크로스페이더 드래그 | **마우스 input이 래치 해제(normal 복귀)**. (슬라이더 비활성화 대신 — 마우스 우선 원칙 보존) |
| A4 | 래치 중 커브 변경 | 커브 버튼도 `applyCrossfade` 경유 → 컷값을 새 커브로 재적용(immediate). 직접호출 제거 |
| A5 | 양 덱 다른 모드 | Pointer Lock은 document 전역 단일 → **한 번에 한 덱만 락**. 다른 덱 락 요청 시 기존 락 해제 후 전환 |
| A6 | rAF가 래치 컷을 되돌림 | latched 동안 rAF cfTarget 추종 스킵(§3.2) |
| B1 | 락 스크래치 중 pointerup | **종료 아님**(락 유지, 이동으로 계속). precision cleanup에서 'up' 제외 ← **양손 급소** |
| B2 | 락 중 Shift 부작용 | scratchCurve/movementY는 modifier 무시. Shift는 CF 라우팅만 소비 |
| B3 | 락+패널+래치 Esc 경합 | **락 활성이면 Esc=락 해제만**(패널/래치 불변). pointerlockchange로 판정, ESC keydown과 우선순위 분리 |
| C1 | 래치 vs 핫큐 Shift | Shift+CF키=래치토글, Shift+핫큐키=삭제. e.code 직교(Q/W/E ∩ ASDF/ZXCV=∅) |
| C2 | 락/래치 vs R·Space롤·reverse | 차단 안 함(독립 분기). 회귀 체크: 락 중 Space롤·R 토글 동작 |
| C3 | 락 커서숨김 vs 플래터 시각 | precision은 `deck.dragging` 미사용 → curVel 기반 회전 경로 별도(updateDeckVisual :2157이 안 멈추게) |
| 디스패치 | Shift없는 Q를 normal/latched 어디로 | 모드 먼저 검사 후 키 처리(§3.2 규칙). 컨트롤러가 단일 디스패치 |

## 8. UX / 시각 피드백 (필수)
- **Pointer Lock 진입 토스트**(커서 실종 대비): "정밀 모드 · Esc 해제" 1회. (선택 아님 — 필수)
- **모드 뱃지**: 덱별 Vinyl/Precision 표시(기존 engineMode 패턴 재사용).
- **래치 활성 표시**: cf-keys(Q W E)·레일에 앵커 키 하이라이트.
- **모바일/미지원**: Precision 토글 숨김+Vinyl 고정, 래치는 키보드 없으면 비노출(feature-detect).
- **락 중 제약 문서화**: 마우스가 한 덱에 갇혀 EQ/반대덱 마우스 조작 불가 → "락 풀고 조작" 안내(EQ 키매핑은 백로그).
- **영속(범위 결정)**: 최소 `scratchMode`·`precisionSens` localStorage 저장 권장.

## 9. 구현 단계 (각 동작하는 최소 단위)
**Phase 0 — 안전 인프라 + 진짜 디지퍼 + 게인 권위 일원화**
- `setParam` 헬퍼. **디지퍼 대상 = 슬라이더 input(:2055)·EQ/볼륨(:1746-1749)·커브(:2067)** ('smooth'). **applyCrossfade는 즉시대입 유지**(건드리지 않음). `ensureNode`(:1629)·커브를 applyCrossfade 경유로 통일(gainA/B read-only화).
- `sanitizeVel` 메인 게이트 + worklet/fallback NaN 방어(:727/:982/:854 동기). dt floor.
- `latencyHint:'interactive'` + baseLatency 로그.
- 검증: node 문법 파싱(`node --check` 동등) + NaN 주입 단위테스트 + EQ/슬라이더 드래그 클릭 청취.

**Phase 1 — ScratchInput 추출 + 플래터 mode 훅 (동작 보존)**
- 공용 `endScratch`/`emit`/`sanitize` 추출, source별 mapping/state 분리 유지. **:1791/:1805/:1832를 mode-aware로**(precision 훅 자리 + lock 중 window onMove early-return). visualAngle 계약 명시. blur/visibilitychange 종료 추가.
- 검증: **"platter 드래그 중 판이 손 따라 도는가"** 포함 회귀 체크리스트.

**Phase 2 — Precision 스크래치 (Pointer Lock)**
- 모드 토글 UI+feature-detect. 락 진입(raw+폴백, buffer가드 후)·`pointerlockchange` 관리·document onMove(movementY→정규화→scratchCurve)·진입 토스트·모드 뱃지·시각 회전 경로·민감도 슬라이더. **pointerup은 락 종료 아님.**
- 검증: Chromium(raw)/FF(폴백) 실측, Esc 재진입, **F1 양손 동시 PASS**.

**Phase 3 — CrossfaderController 추출 (동작 보존)**
- normal 캡슐화(cfKeyStack/이징/마우스우선). 키 라우팅→컨트롤러. crossfade 단일 권위.
- 검증: 크랩·단타우선·마우스우선 회귀.

**Phase 4 — 래치 컷 (latched)**
- Shift+Q/W/E 탭 토글·`anchorValueFromGeometry`(DOM-relative, resize 재계산)·Q/W 펄스(PULSE_MS)·E 긴컷·컷=setParam('cut') 이징 bypass·래치 시각표시·CF_CUTLAG. rAF 추종 스킵(§3.2).
- 검증: 펄스가 이징에 안 빨림, E 긴컷, A3/A4 정책, Shift분기.

> 순서: Phase0(안전망+권위) → Phase1(추출+precision 훅 자리) → Phase2(락) / Phase3(CF추출) → Phase4(래치). 추출과 기능 분리로 회귀 격리.

## 10. 블로킹 리스크
B1 NaN→worklet 오염(sanitize 메인+worklet 이중, dt floor) · B2 fallback 가드 누락(메인 공통부+짝 grep) · B3 락 해제 Esc↔패널 ESC(이벤트로만, 우선순위) · B4 종료 시 scratch-end 누락(단일 endScratch) · B5 래치컷↔rAF(이징 bypass+추종 스킵) · B6 gain 권위(applyCrossfade 단일 라이터, :1629 교체) · **B7 Precision pointerup 종료 모순(up 제외)** · **B8 applyCrossfade에 ramp 금지(즉시대입 유지)**.

## 11. 검증 방법 (빌드 없음)
- 문법: node 파싱(`node --check` 동등).
- 단위: sanitizeVel(NaN/Inf/clamp) · scratchCurve(단조·경계·비NaN) · anchorValueFromGeometry · worklet NaN주입→position 비오염 · **PULSE_MS ≥ declick fade×2 상수관계**.
- 회귀 체크리스트(Phase1/3): 각속도·**드래그중 판회전**·세로스크럽·탭seek·관성·크랩·단타우선·마우스우선·락중 Space롤/R.
- **F1 (필수 PASS)**: 락 스크래치 진행 중 Shift+Q→W연타→E홀드가 스크래치 안 끊고 컷 들어감 — 1인 양손 수동.
- F2: Pointer Lock 실측(콘솔 lock state 로그) Chromium raw / FF 폴백.
- 브라우저 음질: 컷 클릭·precision 체감·baseLatency. (http 필수: `python -m http.server 8000`.)

## 12. 범위 밖
SAB(②)·worklet 대수술·그리드 bake(④)·죽은코드(⑥⑦)·앵커 빨간선 렌더(DOM 오버레이 권장, 캔버스 금지)·unadjustedMovement 미지원 캘리브레이션 UX·EQ 키매핑·튜너블 기본값 실청 확정.

## 13. 변경 이력 (리뷰 반영 — 정직성)
당초 "C 풀 추출" 선택했으나 적대적 리뷰가 결함 발견 → 정제:
- **per-mode smoothing 낮추기 폐기**(0.997 낮추면 zipper 악화) → C의 핵심 차별점 무효 → 실질 **"B + worklet NaN 방어가드"**.
- **applyCrossfade에 setParam 적용 철회**(매 프레임 ramp=zipper/부하) → 즉시대입 유지, 디지퍼는 input 연사처로 재지정.
- **Precision pointerup 종료 → 락해제/토글/Esc만**(양손·한손 가능의 전제).
- ScratchInput 평탄 통합 → source별 분리 유지(platter visualAngle 사이드이펙트 보존).
- gain 권위 일원화·rAF 래치 스킵·다수 엣지케이스 정책(7절)·UX 시각피드백(8절)·검증 F1~F3 추가.

## 14. 참고
Pointer Lock(MDN/W3C 2.0/web.dev raw input) · AudioParam(MDN setTargetAtTime·linearRamp·cancelAndHold·currentTime, alemangui ramp-to-value) · latencyHint(MDN) · UX(VirtualDJ C/B/V+X/N, Serato/rekordbox cut-lag, libinput curve, MDN jog) · 코드매핑 본 리포 index.html.
