# 애매한 모델링 결정

좋은 모델러는 애매함을 없애는 사람이 아니라, **애매함을 알아채고 이유 있게 결정하는 사람**입니다.
하나의 개념이 여러 방식으로 모델링될 수 있을 때, 그 결정을 기록하세요.

최소 요건: **2개 이상.**

## 흔한 예

| 개념 | 가능한 모델링 |
|---|---|
| 승인 (Approval) | State(승인됨) · Transition(승인하다) · 별도 Entity(승인 기록) |
| 위험 평가 (Risk Assessment) | 별도 Entity · Attribute(위험 점수) |
| 약점 (Weak Point) | Entity · Attribute · 파생된 State |
| 법무 검토 (Legal Review) | State(검토 중) · 별도의 Work(담당자 · 마감이 있는 일) |

## 결정 1 — 범위: 사양 확인과 승격을 World 안에 둘 것인가

- **개념:** 사양 확인(설계 기준서 approve)과 정본 승격
- **고려한 대안:**
  - 범위를 "빌드 → 검증 PASS"로 유지하고, 사양 확인과 승격은 World 밖으로 제외한다.
  - 범위를 유지하되, 사양 확인은 Job의 속성(`spec_approved = true`)으로 두고 `build`의 guard가 확인한다.
  - 범위를 넓혀 사양 확인과 승격을 State로 넣는다.
- **최종 선택:** 범위를 넓혀 State로 넣는다 (`spec_drafted → spec_approved → … → verified → promoted`).
- **이유:**
  - 금지 규칙 4개(셀프 검증, 사양 미확인 빌드, 미수렴 모델 검증, PASS 없는 승격)를 모두 World에서 막고 싶었다. 규칙이 걸린 단계가 범위 밖에 있으면 World가 막을 수 없다.
  - 사양 확인과 승격은 누가 했는지가 중요하다. 특정 역할만 해야 하므로 역할이 붙은 전이로 두어야 한다.
  - 속성으로 두면 확인하는 행위가 World 밖에서 일어나, 누가 그 값을 바꿨는지 World가 검사할 수 없다.
- **ESTC에 미친 영향:**
  - Entity: 기준서(`DesignBasis`)와 카탈로그(`Catalog`)가 Job이 참조하는 Entity로 들어왔다.
  - State: `spec_drafted`, `spec_approved`, `promoted`가 추가되어 상태가 6개가 되었고, `promoted`가 terminal state가 되었다.
  - Transition: `approve_spec`(spec_approver)과 `promote`(curator)가 추가되었다.
  - Constraint: 사양 미확인 빌드와 PASS 없는 승격은 별도 Constraint 없이 `build.from = spec_approved`, `promote.from = verified`로 막힌다.

## 결정 2 — 수렴: 상태로 둘 것인가, 그리고 무엇으로 수렴을 판정할 것인가

- **개념:** EBSILON 계산의 수렴
- **고려한 대안:**
  - (표현 방식) State(`converged`) / Attribute(수렴 여부·오차) / 둘 다
  - (판정 값) 열수지 오차 % 임계값 / 참·거짓(`solver_converged`) / EBSILON 결과 코드(`calc_status`)
- **최종 선택:** 수렴은 State(`converged`)로 두고, `converge` 전이의 guard가 EBSILON 결과 코드 `calc_status ∈ {0, 1}`을 확인한다 (0 Success, 1 SuccessWithWarnings).
- **이유:**
  - `verify`가 `converged`에서만 시작하므로, 미수렴 모델 검증 요청을 별도 규칙 없이 상태 순서만으로 막을 수 있다.
  - FAIL로 `built`에 돌아가면 반드시 `converged`를 다시 거쳐야 하므로, 재계산 없이 재검증하는 일을 막는다.
  - 작업 건이 계산을 끝냈는지가 상태 하나로 드러나, 사람과 에이전트가 같은 단계를 본다.
  - 수렴 허용오차는 EBSILON 입력값에 이미 적용되므로, World가 같은 기준을 % 임계값으로 다시 둘 필요가 없다.
  - 참·거짓은 누군가 판단해서 적는 값이지만, 결과 코드는 EBSILON이 실제로 돌려준 도구 출력이다.
  - 수렴·with warning·미수렴(최대 반복 도달, 오류)을 코드로 구분할 수 있어서 허용 범위를 명확하게 정할 수 있다.
- **ESTC에 미친 영향:**
  - Entity: `ModelFile`에 속성 `calc_status: number`(EpCalculationResultStatus)가 생겼다.
  - State: `converged`가 `built`와 `verified` 사이에 들어갔다.
  - Transition: `converge`(built → converged, modeler)가 생겼다.
  - Constraint: `C2_solver_converged`(`job.model.calc_status in [0, 1]`)가 `converge`의 guard가 되었다.

## 결정 3 — 검증 판정(VERDICT)을 어떻게 표현할 것인가

- **개념:** 검증자의 PASS / FAIL 판정
- **고려한 대안:**
  - 전이로만 표현한다 (`verify` / `reject`).
  - 전이에 더해 `verified_by` 같은 속성을 기록한다.
  - 검증 기록을 별도 Entity로 둔다.
- **최종 선택:** 전이로만 표현한다. PASS는 `verify`(converged → verified), FAIL은 `reject`(converged → built).
- **이유:**
  - 몇 번 검증했는지보다 지금 어느 단계인지가 중요하다.
  - 상태기계를 하나로 유지하고 guard 참조를 단순하게 두어, 작고 정확한 World를 만든다.
  - 누가 검증할 수 있는지는 역할(`verifier`)과 `C1_no_self_verify`가 이미 막으므로, `verified_by`를 따로 기록할 필요가 없다.
- **ESTC에 미친 영향:**
  - Entity: 검증 기록 Entity를 만들지 않았다.
  - State: 판정 결과는 `verified` 또는 `built`로 돌아가는 상태 변화로만 남는다.
  - Transition: `verify`와 `reject` 두 전이가 생겼고, 둘 다 `principal_role: verifier`이다.
  - Constraint: 두 전이 모두 `C1_no_self_verify`(`principal.id != job.modeler`)를 guard로 쓴다.
