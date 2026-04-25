# mergeowl 스펙

> 제품 및 구현 논의를 위한 작업 초안입니다.

## 한 줄 요약

mergeowl는 diff, pull request, patch를 대상으로 하는 장난기 있으면서도 실용적인 AI 리뷰어로, 뻔한 LLM 잡담 대신 엔지니어에게 실제로 도움이 되는 범위 잘 잡힌 리뷰 코멘트를 제공하는 것을 목표로 합니다.

## 문제

코드 리뷰에는 거슬리는 실패 모드가 두 가지 있습니다.

1. **사람은 대역폭이 제한되어 있습니다.** 작은 팀은 루틴한 변경, 큰 diff, 사이드 프로젝트에서 깊은 리뷰를 자주 생략합니다.
2. **현재 AI 리뷰는 종종 노이즈가 많습니다.** diff를 다시 설명하거나, 없는 이슈를 지어내거나, 우선순위가 없는 모호한 제안을 내놓는 경우가 많습니다.

실제 공백은 "모든 코드를 완벽히 이해하는 AI"가 아닙니다. 필요한 것은 다음을 할 수 있는 리뷰어입니다.

- patch를 꼼꼼히 읽고,
- 소수의 고신호 이슈를 짚어내며,
- 왜 중요한지 설명하고,
- 실제 diff에 근거를 두고,
- 모든 변경마다 돌릴 수 있을 정도로 저렴하고 단순해야 합니다.

## 목표

- diff, pull request, pasted patch에 대해 **고신호 리뷰 코멘트**를 생성한다.
- **실행 가능한 지적**을 우선한다: 정확성, 회귀, 보안, 엣지 케이스, 유지보수성, 누락된 테스트.
- 작은 팀이 프롬프트, 평가 데이터셋, 모델 동작을 빠르게 반복할 수 있을 정도로 제품을 단순하게 유지한다.
- 리뷰어 동작이 읽히게 만든다: 코멘트는 구체적 라인/헝크를 참조하고 명확한 이유를 포함해야 한다.
- 다양한 프롬프트/모델/리뷰 정책을 비교할 수 있는 가벼운 실험 환경을 지원한다.

## 비목표

- 완전 자율 코드 승인 또는 merge 권한.
- 아키텍처 결정, 제품 판단, 깊은 도메인 맥락에서 사람 리뷰를 대체하는 것.
- 첫날부터 거대한 엔터프라이즈 워크플로를 모두 커버하는 것.
- 핵심 리뷰어 품질이 나오기 전에 거대한 범용 코딩 에이전트를 만드는 것.

## 사용자

- 모든 patch에 대해 두 번째 눈을 원하는 **인디 해커 / 소규모 팀**
- 들어오는 PR을 빠르게 1차 검토하고 싶은 **메인테이너**
- 리뷰 중심 프롬프트와 평가를 실험하는 **연구자 / 에이전트 빌더**
- PR을 열기 전에 pasted diff에 대한 피드백을 원하는 **개발자**

## 핵심 사용 사례

1. 개발자가 git diff를 제출하면 push 전에 우선순위가 매겨진 잠재 이슈 목록을 받는다.
2. PR이 자동으로 리뷰되고, mergeowl가 짧은 요약과 inline comment를 남긴다.
3. 메인테이너가 같은 patch 세트에 대해 여러 리뷰 프롬프트/모델을 비교한다.
4. 사용자가 patch를 채팅/UI에 붙여 넣고, repo 연동 없이 리뷰 코멘트를 받는다.

## 제품 형태

### 입력

- git에서 생성된 unified diff
- PR patch / changed files 메타데이터
- 선택적 저장소 컨텍스트:
  - 변경된 파일 내용
  - 주변 코드 윈도우
  - 관련 테스트
  - 파일 경로 / 언어 힌트
- 선택적 리뷰 정책/프로필:
  - 엄격하게 vs 느슨하게
  - 버그/보안/성능/스타일 중심
  - 최대 코멘트 수

### 출력

기본 출력은 사람이 읽을 수 있는 렌더링과 함께, 간결한 구조화 리뷰 객체여야 합니다.

권장 구조:

- **summary**: 위험 영역에 대한 2-5개 불릿 개요
- **findings**: 정렬된 코멘트 목록. 각 항목은 다음 포함:
  - severity: blocker / warning / nit
  - confidence
  - file + line/hunk reference
  - claim: 무엇이 잘못되었거나 위험한지
  - why: 짧은 이유
  - suggestion: 구체적인 수정 또는 확인 제안
- **uncertainties**: 모델 확신이 낮고 더 많은 컨텍스트를 원하는 지점
- **pass verdict**: 큰 문제 없음 / 주의 필요

### 리뷰 동작

mergeowl는 다음을 지향해야 합니다.

- 장황한 출력보다 **적지만 더 좋은 코멘트**를 선호한다.
- 변경된 코드에 근거를 두고, 가용 증거를 넘는 추측을 하지 않는다.
- 다음을 명확히 구분한다:
  - 가능성이 높은 버그,
  - 잠재적 리스크,
  - 스타일 제안.
- 가치가 없으면 뻔한 diff 요약을 반복하지 않는다.
- 변경이 실제 동작을 의미 있게 바꿀 때만 테스트 누락을 강하게 지적한다.
- diff에 컨텍스트가 부족하면 불확실성을 인정한다.

좋은 코멘트는 다음과 같아야 합니다.

- 구체적이고,
- 반증 가능하며,
- 코드에 직접 연결되어 있고,
- 엔지니어가 30초 안에 유용성을 느낄 수 있어야 한다.

## 시스템 설계 노트

### 모델 전략

단순하게 시작합니다.

**기준선:**
- 강한 범용 LLM 위에 하나의 강한 프롬프팅 파이프라인
- diff chunking + 소량의 파일 컨텍스트
- finding용 구조화 출력 스키마

**단기 확장:**
- 2단계 리뷰:
  1. 후보 finding 생성
  2. 비평 / 중복 제거 / 랭킹
- 흔한 버그 패턴을 위한 언어별 휴리스틱
- 명백히 저위험 diff를 건너뛰는 저비용 prefilter
- 고심각도 코멘트를 위한 선택적 2차 verifier 모델

**중요 제약:**
이것은 사이드 프로젝트 친화적인 시스템이므로, 복잡한 에이전트 스택보다 반복 속도가 더 중요합니다. 정교한 오케스트레이션보다 강한 기준선과 좋은 평가가 우선입니다.

### 모델 타깃 상태

현재 논의는 유용한 **비교 사다리(comparison ladder)** 에는 수렴했지만, 최종 모델 타깃 자체는 아직 고정되지 않았습니다.

**결정된 것:**
- mergeowl는 단일 기준선이 아니라 단계적인 Qwen 사다리를 상대로 평가한다.
- `Qwen2.5-Coder-32B-Instruct`는 **anchor / 하한 baseline** 역할이다.
- `Qwen3.5-27B`는 코딩 성능 기준 **첫 번째 본격 타깃**이다.
- `Qwen3-Coder-*`, `Qwen3.6-*`는 **상단 reference 모델**이며, day-one 성공 기준은 아니다.

**아직 미정인 것:**
- mergeowl 파라미터 수 / active parameter budget
- 베이스 아키텍처 계열
- 목표 컨텍스트 길이
- 목표가 다음 중 무엇인지:
  - 비슷한 규모에서 `Qwen3.5-27B`에 근접 / 동등 성능을 내는 것인지,
  - 더 작고 / 더 저렴한 모델로 비슷한 코딩 품질을 내는 것인지

이 차이는 스펙을 순수 capability 타깃으로 볼지, capability-efficiency 타깃으로 볼지를 바꿉니다.

### 평가

평가는 **recall보다 precision을 우선**해야 합니다.

이유: 시끄러운 리뷰어는 금방 mute 됩니다.

권장 평가 슬라이스:

- 실제 과거 PR과 알려진 리뷰 코멘트
- synthetic bug-injected diff
- 깨끗한 리팩터 / formatting-only diff
- 테스트만 바뀐 변경
- 의존성 / 설정 변경
- 보안 민감 patch

#### 모델 벤치마킹 전략

모델 비교 작업에 대한 현재 스펙은 다음과 같습니다.

**Phase 0 — anchor baseline**
- `Qwen2.5-Coder-32B-Instruct`

**Phase 1 — primary target**
- `Qwen3.5-27B`

**Phase 2 — stronger references**
- `Qwen3.5-35B-A3B`
- `Qwen3-Coder-30B-A3B-Instruct`
- 선택 사항: `Qwen3-Coder-Next`

**Phase 3 — upper references / stretch comparisons**
- `Qwen3.6-27B`
- `Qwen3.6-35B-A3B`
- 선택 사항인 API-only reference: `Qwen3.6-Plus`

해석 규칙:
- `Qwen2.5`는 "우리가 더 오래된 하한 baseline은 넘는가?"를 답한다.
- `Qwen3.5`는 "우리가 현재 시점의 실용적인 오픈 타깃과 경쟁 가능한가?"를 답한다.
- `Qwen3-Coder` / `Qwen3.6`는 "더 강한 코딩 특화 / 최신 세대 reference와의 거리가 어느 정도인가?"를 답한다.

#### 벤치마크 롤아웃

의도된 롤아웃은 한 번에 전부가 아니라 점진적 확장입니다.

1. **Pilot:** mergeowl vs `Qwen2.5-Coder-32B-Instruct`, `Qwen3.5-27B` 비교
2. **우선 코딩 벤치 하나만 사용:** 예: LiveCodeBench 또는 HumanEval/MBPP
3. **1차 핵심 메트릭 하나를 먼저 고정:** 예: `pass@1`
4. **Pilot이 안정화된 뒤에만** `Qwen3-Coder`, `Qwen3.6`로 확장

이렇게 해야 초기 비교가 저렴하고, 읽기 쉽고, 해석 논쟁이 덜합니다.

#### 평가 계약

스펙 레벨에서 cross-model 비교는 **동일한 추론 조건** 아래 실행되어야 합니다.

즉 벤치마크 harness는 가능한 한 다음을 고정해야 합니다.
- 프롬프트 형식
- tool / scaffold policy
- decoding 설정
- context budget
- 평가 데이터셋 버전
- 실행 및 채점 규칙

엔진 선택이나 양자화 같은 구현 세부는 아래에서 달라질 수 있지만, 비교 계약 자체는 결과 차이가 평가 조건이 아니라 모델 차이에서 오도록 해야 한다는 것입니다.

#### 메트릭

제품 리뷰 메트릭:
- finding precision (코멘트가 실제로 유용한 비율)
- severe false positive rate
- duplicate comment rate
- grounding quality (코멘트가 실제 diff에 묶여 있는 정도)
- coverage of seeded bugs
- 평가자(reviewers)의 선호도 점수

코딩 벤치 메트릭:
- 선택한 벤치마크에서의 `pass@1`
- 2차 보고 지표로서 latency / cost
- stochastic decoding을 쓸 경우 rerun 간 분산

각 finding에 대한 단순 루브릭:

- correct / incorrect
- important / minor / irrelevant
- grounded / weakly grounded / hallucinated
- actionable / vague

#### 성공 기준 상태

방향성은 합의되었지만, 정확한 수치 목표는 **아직 열려 있습니다**.

모델 작업이 스펙상 완료로 간주되기 전에 다음을 고정해야 합니다.
- mergeowl 모델 크기 / 아키텍처 타깃
- 정확한 1차 코딩 벤치마크
- 정확한 성공 임계값, 예:
  - `Qwen3.5-27B` 대비 `pass@1` ±X% 이내, 또는
  - 절대값 기준 `pass@1 >= Y%`
- phase 확장 트리거, 예:
  - Phase 1 타깃을 달성하면 `Qwen3-Coder` / `Qwen3.6` 비교로 확장
  - 아니면 평가 범위를 넓히기 전에 모델을 먼저 반복 개선

### 서빙 / 운영

초기 시스템은 저렴하고 쉽게 돌릴 수 있어야 합니다.

- PR 또는 수동 diff 제출 시 트리거
- diff chunk를 결정적으로 계산
- 변경된 파일에 대해서만 최소 repo context 조회
- 엄격한 스키마로 리뷰 모델 호출
- 평가 및 회귀 테스트를 위해 raw input/output 저장

운영 우선순위:

- 재현 가능한 프롬프트 / 설정
- PR당 리뷰 비용
- 개발자 워크플로에 맞는 latency
- 예전 diff를 새 프롬프트/모델로 쉽게 replay 가능해야 함
- 왜 해당 코멘트가 생성됐는지 감사 가능성(auditability)

## 열린 질문

- mergeowl는 먼저 **GitHub PR 리뷰**에 최적화해야 하나, 아니면 **standalone diff/paste review**에 최적화해야 하나?
- 비용/지연이 과해지기 전에 repository context가 precision을 얼마나 개선하는가?
- 이 도메인에서 2단계 리뷰어가 one-pass prompting보다 실질적으로 더 나은가?
- 적절한 기본 코멘트 예산은 3, 5, 10 중 무엇인가?
- MVP에서 "nit" 코멘트를 둘 것인가, 아니면 warning/blocker만 둘 것인가?
- 평가/UX 일관성을 위해 day one부터 안정적인 JSON 리뷰 포맷이 필요한가?

## 제안 MVP

실용적인 MVP는 다음과 같습니다.

1. unified diff를 입력받는다.
2. 변경된 파일 snippet만 가져온다.
3. 고정된 스키마로 최대 5개의 finding을 생성한다.
4. 이를 간결한 리뷰 요약으로 렌더링한다.
5. 예시를 offline evaluation용으로 저장한다.

이 정도면 첫 번째 진짜 질문에 답하기 충분합니다.

**mergeowl가 엔지니어가 일관되게 유용하다고 느끼는 코멘트를, 감내 가능한 수준의 낮은 false positive와 함께 생성할 수 있는가?**
