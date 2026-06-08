# 코드 리뷰 & 수정 리포트 — Vinyl Scratch Engine

> **Round 1 리뷰**: 2026-06-08 · 멀티 에이전트(37 agents, Opus 4.8, ~164만 토큰, ~10분)
> **대상**: `index.html` (리뷰 당시 2234줄) · 8개 차원 리뷰 → 결함별 **적대적 검증**(default-refute) → completeness critic
> **결과**: 28건 제기 → **25 확정 / 3 기각** + critic 추가 갭 16건
> **원칙**: 모든 수정은 worklet↔fallback **DSP parity 보존**. 검증 = `node`로 `new Function()` 문법 체크 + `detectBPM` mock 단위 테스트.
> 작업 브랜치: `review-fixes` (base `e75c0d5`)

---

## ✅ 적용 + 검증 완료

| 커밋 | 클러스터 | 핵심 |
|------|----------|------|
| `6b5a544` | dead loop 제거 | `handleLoop` 호출 0회 — 루프 패드가 roll로 대체돼 고아. main+worklet+fallback+HTML/CSS 대칭 제거 (-71줄) |
| `d6b0f56` | BPM 검출 | comb `/=cnt`(느린편향)→**`/=sqrt(cnt)`(octave 균형)**, 포물선은 **실제 local-max일 때만**+offset±0.5 클램프, 결과 **[60,200] 검증→실패 시 null** |
| `a5df1da` | 타임스트레치 | `pushBuffer(buf, preserve)` **재생위치·핫큐 정규화 remap**(load 후 seek 재전송), ratio 비례 **cap**(절단 제거), **lib 미로딩 피드백**, **mono** 유지 |
| `3ff2749` | 스크래치 | **held-platter 50ms idle 워치독**(잡고 멈추면 정지), flick **velocity ±8 클램프**(시각은 1:1) |
| `ff892d8` | 로드/lifecycle | **loadSeq 레이스 토큰**, **XSS escape**(`escHtml`), 디코드 실패 시 덱 리셋, **BPM-null 잔존 제거**, **init brick 해제**, `ensureRunning()` resume |
| `238f10a` | 키보드 | **`spaceRollActive`** 대칭(keyup 오발 방지), **blur/visibilitychange panic 리셋**(stuck 키 해제) |
| `2d35fe5` | DSP | 'load' 시 **roll 상태 리셋**(stale 구간 정지) + **velocity denormal snap** — 양 엔진 |

### 확정 결함 → 상태 (심각도 = 검증 후 조정치)

**Medium**
- 타임스트레치가 재생위치/핫큐 초기화 → ✅ 보존 remap (`a5df1da`)
- 잡은 채 멈춘 플래터가 마지막 속도로 계속 재생 → ✅ idle 워치독 (`3ff2749`)
- BPM comb가 빠른 트랙을 절반템포로 오검출 → ✅ sqrt 정규화 (`d6b0f56`)
- BPM 포물선 외삽 → 음수/거대 BPM → ✅ local-max 가드+범위검증 (`d6b0f56`)
- loadFile 레이스(늦은 디코드가 새 트랙 덮음) → ✅ loadSeq 토큰 (`ff892d8`)
- 파일명 raw HTML 주입(XSS) → ✅ escHtml (`ff892d8`)
- Space keyup 미가드 → roll 꼬임 / blur 시 stuck → ✅ spaceRollActive + panic (`238f10a`)
- [critic] init 실패 시 양 덱 brick → ✅ audioInitPromise 해제 (`ff892d8`)
- [critic] detectBPM null 시 옛 BPM 잔존 → ✅ null 처리 (`ff892d8`)

**Low**
- timestretch cap 절단(ratio<0.5) → ✅ (`a5df1da`) · soundtouch 미로딩 무피드백 → ✅ (`a5df1da`)
- AudioContext가 Play/R에서 resume 안 함 → ✅ `ensureRunning()` (`ff892d8`)
- 버퍼 reload 시 roll 미해제 → ✅ (`2d35fe5`) · 디코드 실패 시 옛 트랙 생존 → ✅ (`ff892d8`)
- velocity IIR denormal → ✅ snap (`2d35fe5`)
- 워크릿/fallback 상태보고 주기 8x 차이 → ⏳ 미적용 (UI 평활도 nit)

**Nit**
- ✅ mono dual-mono(`a5df1da`), velocity 클램프(`3ff2749`)
- ⏳ 미적용: channelData 배열 hoist(fallback), state-msg 객체 prealloc, `out[0].length` 가드, anti-alias 재진입 transient, Blob URL revoke, detectBPM 범위체크(이미 BPM 커밋에 포함)

---

## 🧪 기각된 3건 (적대적 검증이 잡아냄)
1. **"SVF denormal이 CPU 스파이크"** (×2 차원) — JS에서 Float32 저장값은 읽을 때 normal double로 승격. 실벤치 ~0% 차이. 네이티브 C-DSP 우려의 오이식 → **기각**.
2. **"srcLen/ratio가 SoundTouch latency 무시해 꼬리 절단"** — `soundtouch.js` 확인 결과 silent lead-in 없음(frame 0부터 출력) → **기각**.

---

## ⏳ 미적용 (낮은 가치 / 추가 판단 필요) — Phase 2 후보

**낮은 가치 nit** (위 목록): channelData hoist · state-msg prealloc · out[0].length 가드 · anti-alias 재진입 · Blob URL revoke · 상태보고 8x 주기.

**critic 갭 중 미적용** (대부분 low, 설계 판단 필요):
- per-frame 양 덱 waveform/zoom **무조건 redraw** 비용 (배터리/jank) — dirty 체크 필요
- EQ 노브/roll 패드 **`lostpointercapture` 미처리** (capture 분실 시 dragging 래치)
- 명시적 허용된 **대용량 video** 입력에 크기/길이 가드 없음 → OOM 위험
- **DPR 변경**(모니터 이동) 시 캔버스 backing store 미갱신 → 블러
- `animate()`가 크로스페이더 이징 중 `.value` 덮어써 드래그 stomp 가능
- `analyzeWaveform` IIR 상태가 bin 경계 넘어 carry → 트랙 시작부 Low밴드 warmup 아티팩트
- intro 모달 떠있는 동안 단축키가 뒤 덱에 작동(누수)
- `createDeck`가 window pointer 리스너를 덱별 등록(양 덱 onMove가 매 이벤트 실행 — dragging 가드로 무해하나 결합)

> 이들은 대부분 **저심각·설계 판단** 영역이라 의도적으로 보류. Round 2(맥스 뎁스) 재리뷰 + 사용자 결정 대상.
