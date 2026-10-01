# NewVent

**AI 기반 이벤트 페이지 제작 및 게시 관리 시스템 구현 프로젝트**

이벤트 페이지 하나를 올리려면 보통 기획자가 문서를 쓰고 디자이너가 시안을 잡고 개발자가 퍼블리싱합니다.
NewVent는 그 과정을 **"어떤 이벤트를 하고 싶은지 말하면 페이지가 나오는"** 하나의 화면으로 줄입니다.

```
관리자: "가을 응원 이벤트 만들어줘. 혜택은 쿠폰 3종."
  → LLM이 페이지 생성  → 미리보기  → "제목을 좀 더 크게" → 그 블록만 수정  → 게시
                                                                 ↓
                                                     사용자가 이벤트 페이지에서 참여
```

---
#### >>>>>>>>>> [**10월 2일 금요일 멘토링 질문**](https://github.com/New-Vent/.github/blob/main/metoring.md) <<<<<<<<<<
---

## 저장소

| 저장소 | 역할 | 스택 |
| --- | --- | --- |
| [**newvent-backend**](https://github.com/New-Vent/newvent-backend) | API · LLM 파이프라인 · 검증 | Java 21 · Spring Boot 4.1 · PostgreSQL 17 + pgvector · Flyway |
| [**newvent-frontend**](https://github.com/New-Vent/newvent-frontend) | 사용자/관리자 화면 · 이벤트 페이지 CSS | Vite 8 · 바닐라 JS |
| [**model-benchmark**](https://github.com/New-Vent/model-benchmark) | 어떤 LLM을 쓸지 정한 실험 기록 | Python · Ollama → AWS Bedrock |

---

## newvent-backend

생성·수정·검증·게시가 모두 여기 있습니다. 패키지를 도메인으로 나누고,
**LLM 출력 계약은 `registry` 한 곳에** 모았습니다.

```
src/main/java/com/newvent/
├─ event/          이벤트 CRUD · 버전 · 게시 · 휴지통 · 공개조회
├─ generation/     페이지 생성(템플릿·백지) · 작업 폴링 · 호출 장부 · 일일 상한
├─ editor/         채팅 수정 · 라우터 · 되묻기(ASK_BACK)
├─ registry/       ★ LLM 출력 계약 — Block · Slot · Variant · Palette
│                    검증 · 정화 · 병합 · 프롬프트 빌더
├─ infra/llm/      LlmClient 구현 — mock · ollama · bedrock
├─ auth/           JWT · 사용자/관리자 쿠키 분리 · CSRF
├─ user/ participation/ rag/ admin/ common/
└─ ...
src/main/resources/
├─ db/migration/   Flyway V1~V11
└─ templates/      이벤트 템플릿 5종
docs/              API 규약 · 로컬 설정 · 결정 기록
```

**설계에서 신경 쓴 것**

- **모델 출력을 3단계로 거릅니다** — 정화(script 제거) → 검증(구조·값 보존) → 병합(위치 기반 치환)
- **블록 단위로 수정합니다** — 전체 재생성 대신 라우터가 "무엇을 어디에"를 정하고 해당 블록만 다시 만듭니다
- **못 알아들으면 되묻습니다** — 모델에게 다시 시키지 않고 관리자에게 물어봅니다
- **슬롯은 보여줄 때 채웁니다** — 기간을 저장 시점에 박으면 이벤트 기간만 고쳐도 화면이 안 바뀝니다
- **모든 모델 호출을 장부에 남기고 하루 상한을 겁니다**

## newvent-frontend

사용자 화면과 관리자 화면을 **두 벌로 따로 빌드**합니다.
관리자 번들(편집 스튜디오·채팅)은 무겁고 공개 화면에 내려갈 이유가 없기 때문입니다.

```
src/
├─ user/           사용자 앱  — /         (이벤트 탐색 · 참여)
├─ admin/          관리자 앱  — /admin/   (생성 · 채팅수정 · 버전 · 게시)
├─ shared/         api · repo · dom · 상태 — 두 앱이 같이 씁니다
└─ event-css/      ★ 이벤트 페이지 CSS를 계층으로 나눴습니다
                     00-tokens · 10-fallback · 20-themes · 30-base
                     40-templates · 50-blocks · 60-variants
                     70-components · 90-palettes · 99-a11y
public/assets/     빌드된 event.css — 생성된 페이지가 이걸 링크합니다
deploy/            nginx 설정 · 배포 스크립트 · IAM 정책
scripts/           event-css 빌드
docs/              백엔드 API 계약 · 팀 설정
```

**런타임 의존성이 0개입니다.** 프레임워크 없이 바닐라 JS로 만들었고 Vite는 빌드에만 씁니다.

## model-benchmark

"어떤 모델을 쓸까"를 감이 아니라 **측정으로 정한 기록**입니다.
v1~v7은 로컬 Ollama, v8부터 AWS Bedrock으로 옮겼고 **v11에서 `gemma-3-27b`로 확정**했습니다.

```
base/              공통 엔진 — engine · registry · checks · run
versions/v1~v11/   버전별 실험 — 케이스 · 프롬프트 · 채점기 · README
template/          실험용 템플릿
docs/              방법론 · 버전별 리포트 · 실패 분석
```

**결론** — `gemma-3-27b`가 `claude-haiku-4.5`와 동점인데 **비용이 1/9**였습니다.
생성·수정은 네 모델 모두 만점이라 변별력이 없었고, **갈린 축은 라우터 하나**였습니다.

실험하면서 배운 것도 같이 기록했습니다 — 채점기가 틀려서 모델을 과소평가한 것,
통과율이 거짓말을 한 것, 숫자만 보고 렌더해 보지 않아 놓친 것.

---
