# 여행영어 AI 맞춤 학습 서비스 — 식당·카페 영역 AI 데이터 기획서

> 담당: AI/Data 기획 (식당·카페)  
> 버전: v1.0 / 2026-10-11  
> 이 문서는 AI가 생성한 학습 문장을 사용자에게 제공한다는 사실을 화면에 고지하는 것을 전제로 한다. 실제 매장 정보·가격·영업 여부는 생성하지 않으며, 필요하면 별도 검색 도구의 최신 결과만 사용한다.

## 1. 담당 범위와 목표

여행 중 식당과 카페에서 바로 말할 수 있는 실용 영어를, 사용자의 도시·장소·상황·학습 이력에 맞춰 제공한다. 사용자는 여행지와 상황을 선택하고, AI는 짧은 역할극·핵심 표현·말하기 연습·복습 카드를 만든다.

| 구분 | 설계 내용 |
|---|---|
| 대상 사용자 | 여행 직전 또는 여행 중인 초급~중급 영어 학습자 |
| 해결할 문제 | 문법 중심 학습으로는 주문·알레르기 요청·결제 같은 현장 대화에 즉시 대응하기 어렵다. |
| AI 역할 | 상황 슬롯을 해석해 난이도별 대화문, 표현, 대체 표현, 발음 힌트를 생성한다. |
| AI 기술 요소 | 멀티모달(STT 말하기 평가), Long-term Memory(취약 표현·저장 표현), Agent/워크플로우(상황 분석→생성→검수→저장) |
| 범위 밖 | 실제 맛집 추천, 영업시간·가격 단정, 의료 판단(알레르기 안전 보장), 결제 대행 |

## 2. 데이터 구조

프런트엔드는 아래 정보를 `scenarioRequest`로 전달한다. 개인식별정보는 받지 않고, 사용자 ID는 인증 시스템의 가명 식별자만 쓴다.

```json
{
  "domain": "restaurant|cafe",
  "scenarioId": "R01",
  "country": "US",
  "city": "New York",
  "proficiency": "beginner|intermediate",
  "partySize": 2,
  "dietaryNeeds": ["vegetarian"],
  "memory": {
    "masteredExpressions": ["Could I get ...?"],
    "weakExpressions": ["allergy", "split the bill"],
    "preferredTone": "polite"
  }
}
```

| 필드 | 사용 목적 | 검증 규칙 |
|---|---|---|
| `scenarioId` | 상황별 생성 규칙 선택 | 아래 10개 ID만 허용 |
| `proficiency` | 문장 길이·어휘 난이도 조절 | beginner / intermediate 중 하나 |
| `partySize` | 복수 주문·좌석 대화 생성 | 1~10 정수, 미입력 시 1 |
| `dietaryNeeds` | 요청 표현 선택 | 사용자가 직접 입력·수정, 안전 보장 문구 금지 |
| `memory` | 이미 아는 표현은 축소하고 취약 표현은 반복 | 동의한 사용자만 저장·언제든 삭제 가능 |

### 출력 계약

AI 출력은 UI에서 안정적으로 렌더링할 수 있게 아래 JSON 형식을 따른다. `safetyNote`는 알레르기 관련 상황에만 필수다.

```json
{
  "titleKo": "채식 메뉴를 물어보기",
  "goal": "직원에게 채식 가능 메뉴를 공손하게 확인한다.",
  "dialogue": [{"speaker":"Customer","en":"...","ko":"..."}],
  "keyPhrases": [{"en":"...","ko":"...","use":"..."}],
  "speakMission": "한 문장으로 주문을 요청해 보세요.",
  "followUpQuestion": "...",
  "safetyNote": null
}
```

## 3. 공통 시스템 프롬프트

아래 프롬프트는 서버에 보관하며, 사용자 입력은 구분자 안에 넣어 지시문 오염을 막는다.

```text
당신은 여행영어 학습 코치다. 식당 또는 카페에서 고객이 실제로 말할 수 있는 짧고 공손한 영어만 만든다.

[반드시 지킬 규칙]
1. 제공된 상황 데이터 안에서만 답한다. 존재하지 않는 매장, 메뉴, 가격, 영업시간을 사실처럼 만들지 않는다.
2. 초급은 한 문장 12단어 이하, 대화 4~6턴, 핵심 표현 3개를 쓴다. 중급은 대화 6~8턴, 핵심 표현 4개를 쓴다.
3. 사용자의 masteredExpressions는 새 표현으로 반복하지 말고, weakExpressions 중 관련 표현 하나를 자연스럽게 포함한다.
4. 알레르기·식이요법은 직원에게 확인하도록 표현하고, 안전하거나 안전하지 않다고 단정하지 않는다.
5. 비하·차별·주류의 과음 권유·개인정보 요청을 포함하지 않는다.
6. 영어와 자연스러운 한국어 번역을 함께 제공한다. 발음은 한글 음차 대신 강세가 필요한 단어만 IPA 또는 음절 구분으로 짧게 표기한다.
7. 지정된 JSON 스키마만 반환한다. 마크다운, 설명, 추가 키를 쓰지 않는다.

[상황 데이터 시작]
{{scenarioRequest}}
[상황 데이터 끝]
```

## 4. 상황 10개와 상황별 프롬프트

모든 프롬프트는 위 공통 시스템 프롬프트와 함께 사용한다. `{{...}}` 값은 UI 입력값으로 치환한다.

| ID | 영역·상황 | 사용자 목표 | 상황별 사용자 프롬프트 |
|---|---|---|---|
| R01 | 식당: 예약 없이 입장 | 인원과 대기 가능 여부 묻기 | `식당에 {{partySize}}명입니다. 예약 없이 지금 자리가 있는지, 없다면 얼마나 기다리는지 묻는 역할극을 만들어 주세요.` |
| R02 | 식당: 메뉴 추천 요청 | 선호와 예산을 말하고 추천받기 | `{{preference}}를 좋아하고 1인 예산은 {{budget}}입니다. 직원에게 인기 메뉴 또는 추천을 한 가지 물어보는 역할극을 만들어 주세요. 실제 메뉴명은 만들지 마세요.` |
| R03 | 식당: 식이·알레르기 확인 | 재료와 조리 가능 여부 확인 | `{{dietaryNeeds}} 관련 요청이 있습니다. 재료와 교차오염 가능 여부를 직원에게 확인하고, 확답을 요구하지 않는 공손한 역할극을 만들어 주세요.` |
| R04 | 식당: 주문 변경 | 빼기·추가·맵기 조절 요청 | `주문에서 {{changeRequest}}을 요청하는 역할극을 만들어 주세요. 매장에서 가능 여부를 직원이 확인하는 흐름을 포함하세요.` |
| R05 | 식당: 문제 해결 | 누락·잘못 나온 주문을 정중히 알리기 | `{{issue}} 문제가 있습니다. 비난하지 않고 주문 내용을 확인해 해결을 요청하는 짧은 역할극을 만들어 주세요.` |
| R06 | 식당: 계산과 분할 결제 | 계산서 요청·각자 계산 문의 | `식사를 마쳤습니다. 계산서를 요청하고 {{paymentMethod}} 결제가 가능한지 묻는 역할극을 만들어 주세요. 나라별 관행을 단정하지 마세요.` |
| C01 | 카페: 기본 주문 | 음료 크기·온도·포장 선택 | `{{drinkType}}를 {{size}} 사이즈로, {{temperature}}로 주문하고 포장 여부를 답하는 역할극을 만들어 주세요.` |
| C02 | 카페: 우유·당도·디카페인 변경 | 맞춤 옵션을 공손히 요청 | `{{drinkType}} 주문에서 {{customization}}을 요청하는 역할극을 만들어 주세요. 가능한지 먼저 묻는 표현을 포함하세요.` |
| C03 | 카페: 매장 이용 | 자리·와이파이·콘센트 확인 | `카페에서 잠시 앉아 작업하려 합니다. 좌석 이용, 와이파이 또는 콘센트의 이용 가능 여부를 한 번에 단정하지 않고 물어보는 역할극을 만들어 주세요.` |
| C04 | 카페: 주문 수령·재확인 | 이름 호출·주문 누락 확인 | `주문을 기다리다 {{issue}} 상황을 만났습니다. 주문 번호/이름을 확인하고 정중히 도움을 요청하는 역할극을 만들어 주세요.` |

## 5. 좋은 생성 결과 예시 3개

### 예시 A — R01, 초급, 2명

**입력:** 예약 없음, 2명, 초급  
**좋은 결과의 핵심:** 실제 식당 정보를 꾸며내지 않고, 짧은 질문과 대기 확인을 분리한다.

```json
{
  "titleKo": "예약 없이 2명 자리 묻기",
  "goal": "두 명이 바로 앉을 수 있는지 묻는다.",
  "dialogue": [
    {"speaker":"Customer","en":"Hi, a table for two, please.","ko":"안녕하세요, 두 명 자리 부탁합니다."},
    {"speaker":"Staff","en":"Do you have a reservation?","ko":"예약하셨나요?"},
    {"speaker":"Customer","en":"No, we don't. Is there a table available now?","ko":"아니요. 지금 자리가 있나요?"},
    {"speaker":"Staff","en":"There may be a short wait.","ko":"조금 기다리실 수 있어요."},
    {"speaker":"Customer","en":"That's okay. How long is the wait?","ko":"괜찮습니다. 얼마나 기다리나요?"}
  ],
  "keyPhrases":[
    {"en":"A table for two, please.","ko":"두 명 자리 부탁합니다.","use":"입장"},
    {"en":"Is there a table available?","ko":"빈자리가 있나요?","use":"자리 확인"},
    {"en":"How long is the wait?","ko":"대기 시간이 얼마나 되나요?","use":"대기"}
  ],
  "speakMission":"A table for two, please. 를 말해 보세요.",
  "followUpQuestion":"창가 자리를 원한다면 어떻게 물어볼까요?",
  "safetyNote":null
}
```

### 예시 B — R03, 중급, 견과류 알레르기

**좋은 결과의 핵심:** 교차오염 가능성을 묻되 ‘안전하다’고 보장하지 않는다.

```text
Customer: I have a nut allergy. Could you please check whether this contains nuts?
Customer: Could there be cross-contact during preparation?
Staff: I'll check with the kitchen for you.
Customer: Thank you. I'll wait for confirmation before ordering.

핵심 표현: contains nuts / check with the kitchen / cross-contact
안내: 알레르기 관련 정보는 직원 또는 주방의 최신 안내를 확인한 뒤 주문하세요.
```

### 예시 C — C02, 초급, 디카페인·귀리우유

**좋은 결과의 핵심:** 변경 가능 여부를 먼저 물어보고, 한 주문에 필요한 옵션을 간결하게 담는다.

```text
Customer: Could I get a decaf latte with oat milk, please?
Staff: Sure. What size would you like?
Customer: A small one, please. Is that for here or to go?
Staff: For here or to go?
Customer: To go, please.

핵심 표현: Could I get ...? / decaf / with oat milk / to go
```

## 6. 제외 예시 5개와 제외 기준

아래 결과는 품질 검수에서 재생성 또는 차단한다.

| 번호 | 제외할 출력 예시 | 제외 이유 | 수정 방향 |
|---|---|---|---|
| X01 | “이 식당의 트러플 파스타는 18달러이고 오늘 밤 10시까지 영업해요.” | 제공받지 않은 실제 매장 정보·가격·영업시간을 환각함 | “추천 메뉴가 있나요?”처럼 범용 표현만 생성 |
| X02 | “이 메뉴는 견과류가 전혀 없으니 안심하고 드세요.” | 알레르기 안전을 보장함 | “Could you check with the kitchen?” 및 확인 안내 추가 |
| X03 | “You must give me a free replacement right now.” | 공격적·갈등 유발 어조, 권리 단정 | “I think this may not be what I ordered. Could you help me?”로 교체 |
| X04 | “영수증 사진과 카드 번호를 보내 주세요.” | 불필요한 민감 금융정보 요구 | 주문 번호 또는 주문 내용만 묻고 결제정보는 수집하지 않음 |
| X05 | 초급 사용자에게 15턴짜리 관용구 중심 대화와 어려운 발음 표기를 제공 | 난이도·분량 규칙 위반, 즉시 사용성 낮음 | 초급은 4~6턴·12단어 이하·핵심 표현 3개로 제한 |

## 7. 자동 품질 검수 규칙

생성 후 `validator`가 아래 규칙을 검사한다. 실패하면 같은 입력으로 한 번 재생성하고, 다시 실패하면 일반 템플릿을 반환한다.

| 검수 항목 | 판정 기준 | 실패 처리 |
|---|---|---|
| 스키마 | 필수 키와 문자열/배열 타입 일치 | 재생성 |
| 난이도 | 초급 턴 수 4~6, 핵심 표현 3개, 대화 문장 12단어 이하 | 재생성 |
| 안전 | `guaranteed safe`, 카드번호·여권·비밀번호 요청, 차별 표현 탐지 | 차단 후 안전 템플릿 |
| 사실성 | 제공되지 않은 고유 매장명, 가격·영업시간·주소 단정 탐지 | 재생성 |
| 중복 | memory의 숙지 표현과 핵심 표현 완전 중복률 50% 미만 | 대체 표현 재생성 |
| 언어 | 각 영어 대사에 한국어 번역 존재 | 재생성 |

## 8. Long-term Memory 설계

| 저장 데이터 | 쓰임 | 보관·삭제 원칙 |
|---|---|---|
| 숙지/취약 표현, 최근 상황 ID | 다음 학습의 중복 감소와 복습 추천 | 사용자 동의 후 저장, 설정에서 즉시 삭제 |
| 난이도, 선호 말투, 음성 연습 점수 | 문장 난이도와 피드백 조절 | 원점수 대신 레벨만 저장 가능 |
| 원문 음성·자유 입력 | 기본 저장 안 함 | 녹음 저장은 별도 명시 동의가 있을 때만 |

복습 우선순위는 `취약 표현 → 3일 이상 미복습 표현 → 새 상황 표현` 순서로 정한다. 현지 장소·결제 정보·식이 정보는 대화 맥락에서만 사용하고, 장기 기억에는 사용자가 명시적으로 저장을 선택한 항목만 넣는다.

## 9. 개인 기여 증빙: Git 브랜치·PR 계획

현재 작업 폴더는 Git 저장소가 아니므로, 팀 GitHub 저장소를 받은 뒤 아래 절차로 이 문서를 이관하고 본인 계정에서 실행한다. PR URL과 스크린샷은 발표 자료의 개인 기여 슬라이드에 첨부한다.

```powershell
git checkout main
git pull origin main
git checkout -b docs/restaurant-cafe-ai-data
# docs/restaurant-cafe-ai-data-spec.md 추가
git add docs/restaurant-cafe-ai-data-spec.md
git commit -m "docs: add restaurant and cafe AI data specification"
git push -u origin docs/restaurant-cafe-ai-data
```

PR 제목: `docs: 식당·카페 여행영어 AI 데이터 기획 추가`

PR 본문 체크리스트:

- [x] 상황 10개(R01~R06, C01~C04) 정의
- [x] 공통 시스템 프롬프트와 출력 JSON 계약 정의
- [x] 좋은 예시 3개, 제외 예시 5개 및 판단 근거 작성
- [x] 알레르기·개인정보·환각 방지 규칙 포함
- [x] Long-term Memory와 자동 품질 검수 기준 작성
- [ ] 팀원 1명 이상 리뷰 완료
- [ ] `main` 병합 후 PR 링크를 발표 자료에 기록

권장 증빙 항목: 본인 GitHub 계정의 브랜치, 커밋 SHA, PR 생성일, 리뷰 댓글, 병합 완료 화면. 단순 커밋 수 대신 문서의 담당 범위와 리뷰 반영 내역을 함께 제시한다.

## 10. 인수 기준

- 10개 상황을 각각 선택하면 해당 상황의 학습 결과가 출력된다.
- 식당과 카페가 각각 최소 4개 이상 포함된다.
- 예시 3개는 공통 프롬프트·스키마·안전 규칙을 지킨다.
- 제외 예시 5개는 자동 검수 또는 리뷰에서 걸러지는 근거가 있다.
- 사용자가 ‘저장하지 않음’을 고르면 장기 기억을 읽거나 쓰지 않는다.
- UI에는 “AI가 생성한 학습 콘텐츠이며 실제 매장 정보는 직원에게 확인하세요.”를 표시한다.
