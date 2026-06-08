# 디제이 입력 레이어 리팩토링 플랜 (v3 — 멀티에이전트 재검토 반영)

> 대상: 단일 `index.html` (2291줄, branch `review-fixes`) + `soundtouch.js`. Web Audio 듀얼덱 턴테이블리즘.
> v3 근거: **36-에이전트 재검토**(re-anchor + 6-lens 리뷰 + default-refute 적대검증 27건 + completeness critic). v2 대비 **모든 라인 앵커 재확정 + 17개 수정**.
> v2 원문은 git에 보존: `git show e75c0d5:PLAN.md`.
> **판정: 계획은 본질적으로 타당. Phase 0 즉시 착수 가능. Phase 2(Pointer Lock)는 구현 확정 전 브라우저 스파이크 필요.**

---

## 0. v2 → v3 변경 요약 (재검토 반영)

| # | 출처(검증) | 수정 |
|---|---|---|
| **C1** | DSP-1 ✅confirmed | **NaN 가드는 `scratch-velocity`(worklet 726 / fallback 971)에만.** `scratch-start`/`scratch-end`는 `d.value`를 안 읽음 → 거기 가드 넣으면 `!isFinite(undefined)`→항상 return = **스크래치 진입/종료가 통째로 죽음.** (v2/drift-map의 "start/end에도" 지침은 오류.) 선택적으로 seek(716/962)도 가드 가능. |
| **C2** | R1·overlap | **`sanitizeVel`(±40)는 별도 외곽 NaN/천장 가드.** 기존 platter/zoom **±8 인라인 클램프(1854/1940)는 source-local로 유지.** sanitizeVel이 ±8을 대체하면 anti-spike fix(3ff2749)가 5× 약화. ±40은 precision 경로만 bound. |
| **C3** | DSP-3·core-gain ✅ | **latched 중 rAF 추종블록은 무조건 스킵**(`crossfade!==cfTarget` 가드에 의존 X). 스케줄된 ramp 위에 `.value=`를 쓰면 ramp가 취소됨 → 컷 클릭 재발. **불변식: gainNode에 setParam ramp가 떠 있는 동안 그 노드에 `.value=` 금지.** |
| **C-slider** | DSP-3·DSP-5 | **크로스페이더 슬라이더는 즉시대입 유지**(마우스 우선 + 60Hz applyCrossfade가 스무더). 슬라이더 게인에 setParam-smooth 넣으면 rAF와 race. **디지퍼는 독립 노드인 EQ/볼륨에만**(선택 폴리시, REVIEW가 결함으로 안 본 영역). |
| **C4** | ARCH-4 | crossfade/cfTarget/cfKeyStack **변이 6곳** 모두 컨트롤러 경유: slider(2098-2100)·resetTransientInput(2165-2166)·keydown push(2182)·keyup pop(2200-2202)·rAF(2236-2238)·curve(applyCrossfade). resetTransientInput→`controller.reset()`에 **clearTimeout(pulseTimer) 포함**(A1). 하나라도 옛 변수 직접 변이 남기면 stuck-key(238f10a) 회귀. |
| **C5** | ARCH-1 ✅ | route-through 후 `gainA()/gainB()`(1221-1222)는 **호출처 0 = 죽은코드 → 삭제.** "read-only 화"가 아님. |
| **C6** | ARCH-3 | **ScratchInput은 factory-local**(`createScratchInput(deck,el,…)` createDeck 내부). 모듈레벨 map(`ScratchInput[deck]`) 아님 — 핸들러가 deck/el/zoom 클로저에 의존. 공용은 순수 헬퍼(emit/sanitize/endScratch 계약)만 모듈레벨. |
| **C7** | ARCH-5·PL-3 ✅footgun | **precision은 절대 `deck.dragging`을 set하지 않음**(3곳 불변식: onMove 1832 / onUp 1862 / updateDeckVisual 2214 모두 dragging을 읽음). `deck.lockActive`+`scratchMode` 사용. 위반 시 ①플래터 동결 ②pointerup이 스크래치 종료 ③frozen clientXY로 각도 garbage — 동시 3회귀. |
| **C8** | PL-1·PL-5 | window onMove(1831) **첫 줄 `if(deck.lockActive)return;`**(dragging 체크보다 앞). precision은 **movementX/Y만**(lock 중 clientX/Y 동결). document-level pointermove, `document.pointerLockElement===deck.el.platter`로 게이트. |
| **C9** | PL-2 | `requestPointerLock` **null-guard**: `const p=el.platter.requestPointerLock({unadjustedMovement:true}); if(!p)return; else p.catch(e=>{if(e.name==='NotSupportedError')el.platter.requestPointerLock();})`. Promise형은 비표준(undefined 반환 가능). **lock 부수효과는 전부 `pointerlockchange`에서 구동.** |
| **C10** | PL-4 ✅ | **armIdleStop(50ms→vel0)은 vinyl 전용**(platter+zoom). precision은 arm 금지 — 떼고 간헐 이동이 정상(B1). 안 그러면 50ms마다 stutter. |
| **C11** | DSP-2 | 컷 타깃 = **`cfGains(rail)` 평가값**(하드코딩 0/1 금지). 활성 커브 자동 반영(A4); sharp 커브는 off측이 0이 아니라 ~6.1e-6. |
| **C12** | DSP-4 ✅ | 펄스 양 edge를 **AudioContext 클럭**으로(t0=currentTime; down t0+LA…+CUT_MS; back t0+LA+PULSE_MS…). **F3 검증식 정정: `PULSE_MS > 2·(CUT_MS+PARAM_LA)`** (5ms declick과 무관 — declick은 worklet 내부 재생/정지 엔벨로프). |
| **C13** | B3·EUX ✅high | `:2008` Esc 핸들러는 **동기 유지되는 `lockActive` 플래그**를 보고 `closeAllSettings()` 전에 `if(anyLockActive())return;`. **Phase 2 blocker**(Phase 4 아님). pointerlockchange와 같은 tick에 fire 가능하므로 await 금지. |
| **C14** | EUX ✅high | **§8 UX는 net-new.** 토스트 시스템 없음, `engineMode`는 전역(덱별 뱃지 아님). 토스트 헬퍼·덱별 뱃지·래치 인디케이터 = 새 DOM/CSS. **Phase 2 exit-criteria로 고정**(첫 requestPointerLock 전에 토스트 배선). |
| **C15** | EUX | 모바일 feature-detect = `matchMedia('(pointer:fine)').matches && ('requestPointerLock' in Element.prototype)`. **width 미디어쿼리 금지**(데스크톱 좁은 창 오판). |
| **C16** | EUX·A1 | **래치 앵커 = 토글 시점의 crossfade 스칼라값**(geometry 아님). `anchorValueFromGeometry`는 zoom 전용. 로드 시 `clearPulse()`(load는 resetTransientInput 안 탐). |
| **C17** | A5 | 단일 lock 중재: **같은 tick에 exit+request 금지.** document movementY 핸들러가 `document.pointerLockElement`로 소유 덱 라우팅. |
| **Fold-1** | reconcile·EUX | **intro-modal 단축키 누수**: keydown(2174) set-panel 가드 옆에 `if(introOverlay && !introOverlay.classList.contains('hidden'))return;` 한 줄. **Phase 0.** |
| **Fold-2** | reconcile | **platter `lostpointercapture`→endScratch** 등록(EQ knob 2047 패턴). **vinyl만**(precision은 capture 미사용). Phase 1. |
| **Fold-3** | reconcile | **animate stomp 은퇴**를 Phase 3 명시 acceptance criterion으로(단일권위+normal-return 동기로 공짜). |
| **Crit-1** | critic | `initAudio` AudioContext(:1149)는 옵션 없이 생성 — **`{latencyHint:'interactive'}`+baseLatency 로그는 greenfield/안전**(retry 시 재생성돼 자동 재적용). Phase 0. |
| **Crit-2** | critic | precision은 worklet/fallback 공유 port의 **3번째 고빈도 producer** — fallback port 동작 동일성 Phase 2에서 확인. |

**KEEP DEFERRED**(폴드 금지·스코프 밖): per-deck window-listener 통합 · per-frame redraw · DPR canvas blur · analyzeWaveform warmup · state-report 8x 주기 · 할당 nit · Blob URL revoke · 대용량 video(이미 150MB 게이트).

---

## 1. 한 줄 정의 (불변)
흩어진 크로스페이더·스크래치 입력을 **입력 레이어(`CrossfaderController` + factory-local `ScratchInput`)로 추출**하면서 ①Shift+Q/W/E 래치 컷 ②Pointer Lock 정밀 스크래치를 얹고, NaN 방어가드와 게인 권위 일원화를 회수. **존재이유 = 왼손 컷 + 오른손 정밀 스크래치 양손 동시 연주(core-handedness ✅검증: 키→window, lock→mouse 독립 채널).**

## 2. 검증된 현재 앵커 (v2 라인 전부 stale — 이 표만 신뢰)

| 관심 | v2(stale) | **현재** | 비고 |
|---|---|---|---|
| 메인 keydown 라우터 | :2117 | **2174** (keyup 2197) | CF push 2182 / pop 2200-2202. 옛2117=핫큐패드 click(트랩) |
| ESC 패널닫기(별도 리스너) | :1971 | **2008** | B3 게이트 위치 |
| rAF 크로스페이더 이징 | :2177 | **2234** (animate 2228-2251) | 매프레임 applyCrossfade(2241)=60Hz 스무더 |
| applyCrossfade | :1231 | **1223** | 즉시대입 유지(B8 ✅) |
| ensureNode 게인 직접쓰기 | :1629 | **1595** (fn 1585) | applyCrossfade 경유로 교체 |
| 크로스페이더 슬라이더 input | :2055 | **2097-2102** | cfKeyStack clear 2100, cfTarget 2099. 옛2055=togglePlayBoth 주석(트랩) |
| 커브 스위치 | :2067 | **2105-2110** | 이미 applyCrossfade 호출. 옛2067=Loop-Roll 주석(트랩) |
| EQ/볼륨 input | :1746-1749 | **1735-1738** | knob(2030)이 합성 dispatch로 같은 핸들러 구동 |
| worklet scratch-velocity | :727 | **726** (case 724) | NaN 가드 위치. PROCESSOR_CODE 643~ |
| fallback scratch-velocity | :982 | **971** (case 970) | **옛982=`case 'roll'`(최악 트랩)** |
| position 경계클램프 | :854 | **worklet 858-872 / fallback 1100-1113** | **옛854=roll-wrap 루프(트랩).** 기존 클램프는 NaN 미방어→가드 필수 |
| platter pointerdown | :1791 | **1815** | buffer 가드 1816, dragging set 1819, armIdleStop 1826, setPointerCapture 1827 |
| platter onMove | :1805 | **1831** (window 1868) | 인라인 ±8 클램프 1854, sensitivity=deck.sensitivity(1982) |
| platter onUp | :1832 | **1861** (window 1869-70) | scratch-end 1866. B1/B7 분기 지점 |
| updateDeckVisual | :2157 | **2212** | dragging 게이트 2214 (C3) |
| zoom 스크래치 경로 | — | **pointerdown 1911 / move 1924 / zUp 1946-1960** | scratch-end 1959 — endScratch 통합 대상 |
| initAudio / AudioContext | — | **1149** | latencyHint 추가(Crit-1) |
| resetTransientInput(panic) | — | **2163** (clear 2165-2166, blur 2170 / vis 2171) | controller.reset 대상(C4) |

## 3. 아키텍처 (수정 반영)

### 3.1 모듈 경계
```
keydown(2174)/keyup(2197) ─► CrossfaderController.onKey(code, shift, type)
   ESC(2008) ─► (lock 활성? release : closeAllSettings)            [C13]
pointerdown/move/up, pointerlockchange ─► createScratchInput(deck, el)  [factory-local, C6]
공용 순수 헬퍼(모듈레벨):
  setParam(param, value, {mode:'immediate'|'cut'|'smooth'})        [단일 게인 전이 창구]
  sanitizeVel(v) = isFinite(v)? clamp(v,-SCRATCH_VEL_MAX,SCRATCH_VEL_MAX) : null  [외곽가드, ±8 미대체 C2]
  scratchCurve(normDelta, dt) = 비선형(γ), dt floor
  endScratch(deck) = idempotent (platter+zoom+precision 종료 공용; precision은 pointerup 제외 B1)
```
**입력 라우팅**: 새 리스너 금지(기존 2174·2008·window onMove 재사용). Pointer Lock movementX/Y만 document-level 별도 핸들러.

### 3.2 상태머신
**CrossfaderController** — crossfade의 단일 권위. **6개 변이 사이트 전부 흡수(C4).**
```
onKey(code,shift,type):
  shift & code∈{Q,W,E} & down  → 래치 토글(재탭=해제) [Shift keyup 무시]
  normal & code∈CF_POINTS      → 기존 cfKeyStack 보존(push 2182/pop 2200-2202)
  latched: Q/W down(!repeat) → setParam(gain,'cut',cfGains(반대레일)) + 펄스타이머(C12)
           E down → 컷 유지 / E up → 앵커 복귀
rAF(2234): **latched면 추종블록 전체 무조건 스킵(C3)** — gainNode ramp 중 .value= 금지.
           normal 복귀 시 crossfade=anchor; cfTarget=crossfade; crossfaderEl.value 동기; applyCrossfade.
reset(): cfKeyStack=[]; mode=normal; clearTimeout(pulseTimer); latchKey=null  [resetTransientInput 2165-2166 대체, A1]
```
**ScratchInput (덱별 factory-local)** — source별 mapping/state 분리.
```
idle ─pointerdown(platter,vinyl)─► platter (angular, deck.dragging=true, armIdleStop, window onMove)
idle ─pointerdown(zoom)──────────► zoom    (vertical, zDrag 클로저, 탭seek, armIdleStop)
idle ─pointerdown(platter,precision)+lock─► precision (movementY, document onMove, **dragging=false C7, armIdleStop 금지 C10**)
종료: platter/zoom = pointerup/cancel/**lostpointercapture(Fold-2)** ; precision = pointerlockchange해제/토글/Esc만(B1)
공통: blur/visibilitychange
window onMove(1831) 첫 줄: if(deck.lockActive)return;  [C8]
```

### 3.3 게인/메시지 흐름
- **applyCrossfade(1223)는 유일 게인 라이터·즉시대입 유지.** ensureNode(1595)→applyCrossfade 경유, gainA/B 삭제(C5). 커브(2109)·슬라이더(2101)는 이미 경유.
- **setParam('cut')**(linearRamp)=래치 컷 이벤트만. **setParam('smooth')**(setTargetAtTime ~8ms)=**EQ/볼륨(1735-1738)만**(C-slider: 크로스페이더 슬라이더는 즉시 유지).
- scratch 메시지 재사용. **NaN 가드: worklet 726 + fallback 971 (scratch-velocity 전용 C1, 양엔진 byte-parity).** position NaN 방어: worklet 858-872 / fallback 1100-1113. 기존 denormal snap(777/1021)·±8 클램프 보존.

## 9. 구현 단계

**Phase 0 — 안전 인프라 + 게인 권위 일원화 (✅ 즉시 착수 / critic 승인)**
1. `setParam(param,value,{mode})` 헬퍼: immediate=`.value=`; cut=`cancelScheduledValues(now)`→`setValueAtTime(param.value,now+LA)`→`linearRampToValueAtTime(target,now+LA+CUT_MS)`; smooth=`setTargetAtTime(value,now+LA,0.008)`.
2. 게인 권위: ensureNode 1595 → `applyCrossfade()`; `gainA/gainB`(1221-1222) 삭제. applyCrossfade 즉시 유지.
3. `sanitizeVel` 외곽 게이트(C2, ±8 source 클램프 유지). **worklet 726 + fallback 971 NaN 가드(C1, parity)**; position NaN 방어 858-872/1100-1113.
4. `AudioContext({latencyHint:'interactive'})` + baseLatency 로그(Crit-1, 1149).
5. intro-modal 가드 1줄(Fold-1, 2174). EQ/볼륨 디지퍼(선택 폴리시, 1735-1738).
6. **검증**: `node`로 PROCESSOR_CODE+메인 `new Function()` 문법 / NaN 주입 단위테스트(sanitizeVel·worklet position 비오염) / **worklet↔fallback parity diff**(가드 2줄 양쪽 동일).

**Phase 1 — ScratchInput 추출(factory-local, 동작보존)** · **Phase 2 — Precision(Pointer Lock) ⚠스파이크 게이트** · **Phase 3 — CrossfaderController 추출** · **Phase 4 — 래치 컷**. (세부는 §0 수정표 + §3 적용.)

## 13. Phase 2 선행 스파이크 (구현 확정 전 필수)
브라우저 실측: (a) platter `setPointerCapture`(1827) ↔ Pointer Lock 충돌 — 같은 element에 둘 다 거나? (b) Esc keydown(2008) vs pointerlockchange 발화 순서/동시성(C13 게이트가 동기 플래그로 충분한가). PASS 전 precision DOM/상태 설계 미확정.

## 10. 블로킹 리스크 (재검토 후)
B1 NaN→worklet 오염(가드 726/971+position, scratch-vel 전용 C1) · B5 래치컷↔rAF race(무조건 스킵 C3) · B6 gain 권위(applyCrossfade 단일, 1595 교체) · **B7 precision dragging-false 3중 불변식(C7, 최대 footgun)** · B8 applyCrossfade ramp 금지 · **B9 Esc/lock 중재(C13, Phase2 blocker)** · B10 sanitizeVel↔±8(C2).

## 11. 검증 (빌드 없음)
node 문법 · 단위(sanitizeVel·scratchCurve·worklet NaN주입·`PULSE_MS>2·(CUT_MS+PARAM_LA)` C12) · 회귀 체크리스트(각속도·드래그중 판회전·세로스크럽·탭seek·관성·크랩·단타/마우스우선·락중 Space롤/R) · **parity diff** · F1 양손 동시(필수 PASS) · F2 Pointer Lock 실측(Chromium raw/FF 폴백, lock 전환 A→B).
