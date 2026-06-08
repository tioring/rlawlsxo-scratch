# CLAUDE.md — 디제이 (Vinyl Scratch Engine)

> 브라우저 기반 **듀얼 덱 DJ / 턴테이블리즘** 웹앱. Web Audio API로 밑바닥부터 만든 스크래치 엔진.
> 단일 `index.html` + 라이브러리 `soundtouch.js`.

## 개요
- **무엇**: 2덱 + 크로스페이더 믹서. 가상 LP 판을 드래그해 스크래치, 핫큐/루프/EQ/BPM·타임스트레치.
- **버전**: v0.3 (dual deck)
- **핵심 가치**: **Vinyl(턴테이블) 스크래치가 중심.** CDJ 모드는 폐기됨.

## 실행 (중요)
- **반드시 http로 띄울 것.** `file://`은 AudioWorklet 모듈(Blob URL) 로딩이 막혀 저품질 ScriptProcessor fallback으로 떨어짐.
- 정적 서버: `.claude/launch.json`의 `static-server` = `python -m http.server 8000`.
  - Claude Preview: `preview_start({name:"static-server"})` → http://localhost:8000
- **빌드/패키지 매니저 없음.** 콘솔에 `[engine] AudioWorklet 모드`가 찍히면 정상.

## 아키텍처

### 오디오 그래프 (덱별)
```
scratch-processor(worklet) → EQ(low/mid/high biquad) → volume(gain)
   → crossfade(gain) → masterGain(0.85) → destination
```
- 덱 2개(`deckA`/`deckB`)를 `createDeck(id)` factory로 캡슐화.
- `audioCtx` · `masterGain` · worklet 모듈은 **공유**, 노드 인스턴스는 덱별.

### scratch-processor (`PROCESSOR_CODE`, 인라인 AudioWorklet)
- `position`(float sample)을 `velocity`로 가변속 재생.
- **Hermite 4-point 보간** — 가변속 음질.
- **적응형 anti-alias** — 속도>1일 때 SVF lowpass 2단(cutoff=Nyquist/speed), dry/wet 크로스페이드. `aaEnabled` 토글.
- **declick 게인 엔벨로프** — 재생/정지/점프 시 ~5ms fade. `pendingSeek`로 declick 점프(cue/seek/핫큐 공용).
- **루프** — `loopOn`/`loopStart`/`loopEnd`, position wrap.
- 메시지: `load·play·reverse·rate·cue·seek·scratch-{start,velocity,end}·aa·loop`.
- **ScriptProcessorNode fallback**(`createScriptProcessorFallback`) — worklet 미지원 시 동일 로직.
  - ⚠️ **DSP를 고치면 worklet(PROCESSOR_CODE)과 fallback 양쪽을 반드시 같이 수정.**

### 메인 스레드 기능
- **스크래치**: platter 드래그 각속도 → playbackVel (1바퀴/초 = 1.5x). 손 놓으면 관성(`VINYL_SMOOTHING`).
- **크로스페이더**: 커브 smooth(equal-power)/linear/sharp(tanh). 키 `Q W E R` = 4포인트, 거리비례 속도(지수 ease `CF_TAU`).
- **핫큐**: 덱별 4슬롯, declick seek 점프, progress 마커. 키 `A S D F`(덱A)·`Z X C V`(덱B), Shift=삭제.
- **루프**: In/Out 수동 + 1·2·4 beat(BPM 기반) + Exit. progress에 구간 음영.
- **BPM 자동검출**(`detectBPM`): 200Hz onset envelope → autocorrelation → **comb(하모닉 정규화)** → parabolic 보간. octave 모호성 있음 → 수동 보정칸 제공(베스트에포트).
- **타임스트레치**(`timeStretch`): SoundTouch **오프라인 렌더링**. 원본 → 음정유지·템포변경 buffer → 엔진 재로드(Apply 방식). 끝부분 손실은 0.5초 무음 패딩 + `srcLen/ratio` 길이 보정으로 해결.

## 키맵
```
Q W E R   크로스페이더 4포인트 (현재 위치에서 먼 곳일수록 빠르게)
A S D F   덱 A 핫큐 1~4   (Shift = 삭제)
Z X C V   덱 B 핫큐 1~4   (Shift = 삭제)
Platter 드래그 = 스크래치 · 손 놓으면 관성 복귀
```
(덱별 play/cue 키는 아직 없음 — 버튼으로 조작. 추후 매핑 여지.)

## 핵심 튜닝 상수
- `VINYL_SMOOTHING = 0.9997` — 관성(1에 가까울수록 오래 돔, ~70ms tau).
- `CF_TAU = 0.03` — 크로스페이더 키 추종 속도(작을수록 칼같이).
- anti-alias `maxFc = 0.45·sr`, declick `~5ms`, `masterGain = 0.85`.

## 주요 결정 이력
- **anti-alias = 실시간 가변 cutoff SVF**: 스크래치 가변속에선 고정 BiQuad 부적합, polyphase는 과함.
- **타임스트레치 = SoundTouch 오프라인**: `SoundTouchNode`가 자체 buffer 소스라 우리 scratch worklet과 **실시간 체인 불가** → 오프라인 렌더 후 재로드로 우회(스크래치/루프/핫큐 보존).
- **CDJ 폐기 · Vinyl 고정** — 관성이 핵심.
- **라이브러리만 별도 파일** — 단일 index.html 선호하되 25KB SoundTouch는 `soundtouch.js`로 분리.

## 프로젝트 규칙
- **단일 파일 우선**(index.html). 외부 라이브러리만 별도.
- **DSP 변경 = worklet + fallback 양쪽 동기.**
- **검증**: 빌드가 없으니 `node`로 `<script>`/`PROCESSOR_CODE`를 추출해 `new Function()` 문법 체크 + 알고리즘 단위 테스트(BPM·timeStretch는 mock AudioBuffer로 길이/NaN 확인). 실제 음질은 브라우저에서.

## 의존성
- **`soundtouch.js`** — SoundTouchJS v0.3.0 (LGPL-2.1). ESM `export` 구문만 제거해 클래식 스크립트로 로드(전역 렉시컬에 클래스 노출). 사용 클래스: `SoundTouch` · `SimpleFilter` · `WebAudioBufferSource`. 라이선스 헤더 유지할 것.
