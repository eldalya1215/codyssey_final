# 카테고리 계약 검증 보고서

- 검증 담당: 이주란
- 실행일: 2026-10-11
- 대상: `agent_contract/categories/{restaurant,transport,lodging}.json`
- 범위: 카테고리 계약의 정적 검증 및 실패 사례 검증

## 결과 요약

| 항목 | 결과 | 근거 |
|---|---:|---|
| 실패 사례 검증 | 15 / 15 통과 | 카테고리별 5개 × 3개 카테고리 |
| 카테고리 구조·규칙 검증 | 3 / 3 통과 | `validate_category.py` |
| 검증기 설정 단위 테스트 | 8 / 8 통과 | `test_validate_category.py` |

## 확인한 실패 유형

모든 카테고리에서 아래 다섯 유형이 기대한 코드로 검출됐다.

| 실패 유형 | 기대 코드 | 심각도 |
|---|---|---|
| 금칙 주제·위험 행동 | `forbidden_topic` | block |
| 카테고리에 없는 상황 | `situation_not_in_config` | block |
| 15단어 초과 | `too_long` | warn |
| 영문 문장 중복 | `duplicate_en` | warn |
| 무례한 명령조 | `not_polite` | warn |

## 오탐 점검 및 반영

정상 문장이 부분 문자열 규칙 때문에 `not_polite`로 분류될 위험을 점검했다.

- 식당·카페: `bring me the` 제거 (PR #36, 병합 완료)
- 교통·공항 및 숙소·체크인: `give me the` 제거, 교통의 `get me there` 제거 (PR #39)

따라서 `Could you give me the schedule, please?`, `Could you get me there by seven, please?`처럼 정중하고 실제 여행 상황에서 쓰이는 요청을 차단하지 않는다.

## 실행 명령

```powershell
cd backend
python scripts/check_negatives.py
python scripts/validate_category.py ../agent_contract/categories/transport.json
python scripts/validate_category.py ../agent_contract/categories/lodging.json
python scripts/validate_category.py ../agent_contract/categories/restaurant.json
python -m unittest scripts/test_validate_category.py
```

## 해석 범위

이 결과는 계약 파일과 검증 로직이 정적 규칙대로 동작한다는 증거다. 실제 Gemini 생성 품질, 배포 환경의 API 응답률, 사용자 사용성은 별도의 실환경 측정 및 사용자 테스트로 확인해야 한다.
