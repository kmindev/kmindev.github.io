---
title: "Claude Code effort 설정하기"
date: 2026-08-22 00:00:00 +0900
categories: [AI Agent]
tags: [Claude, Claude Code, Effort, Ultracode, AI Agent, 업무효율화]
---

## 배경

이번 포스팅에서는 effort라는 개념을 정리하고, effort를 적절하게 설정하는 방법들에 대해서 정리하겠다.

## effort란?

effort란 모델이 얼마나 생각할지를 결정하도록 하는 것이다.

effort가 낮으면 간단한 작업을 빠르고 저렴하게 할 수 있으며, 높으면 복잡한 문제에 대해 더 깊은 생각을 하며 추론한다.

| 수준 | 한 줄 요약 |
|---|---|
| `low` | 간단·단순 작업 전용. 제일 빠르고 저렴 |
| `medium` | 비용 아끼려고 지능을 약간 포기 |
| `high` | 기본값. 속도-지능 밸런스 |
| `xhigh` | `high`보다 더 깊게 생각, 토큰도 더 씀 |
| `max` | 가장 깊게 생각하지만 과할 수 있음. 정말 어려운 작업에만 |
| `ultracode` | 생각 깊이가 아니라 작업 처리 방식을 바꾸는 별도 설정 (아래 별도 설명) |

- max는, 복잡한 문제를 풀 때 더 깊은 생각을 한다 → 토큰 소모량이 크다.
- 반대로 low는 간단한 작업에서 사용하며, 빠르다 -> 토큰 소모량이 작다.

## 모델 별 effort 설정

사용 가능한 effort 수준은 모델 별로 약간씩 다르다.

| 모델 | 지원 수준 |
|---|---|
| Fable 5 | `low`, `medium`, `high`, `xhigh`, `max` |
| Opus 5, Sonnet 5, Opus 4.8, Opus 4.7 | `low`, `medium`, `high`, `xhigh`, `max` |
| Opus 4.6, Sonnet 4.6 | `low`, `medium`, `high`, `max` |

Opus 4.6, Sonnet 4.6은 xhigh를 지원하지 않는다.

모델이 지원하지 않는 effort 수준을 선택하면, 설정한 수준 이하의 가장 높은 수준으로 폴백한다.

> 예) Opus 4.6에서 `xhigh`를 선택하면 `high`로 실행한다.

### effort 설정 방법 및 예시

```bash
/effort {level}    # low, medium, high, xhigh, max
/effort high       # 예시

/effort auto       # 모델 기본값으로 초기화
/effort            # 인자 없이 실행하면 슬라이더로 선택

claude --effort {level}   # 시작할 때 해당 세션에만 1회 적용
claude --effort high      # 예시
```

## ultracode

생각하는 깊이가 아니라, 작업의 처리 방식을 결정하는 설정이다.

- `xhigh` 레벨로 추론하고, 동적 워크플로우를 함께 제공한다.
- 동적 워크플로우란 서브에이전트를 대규모로 조율하는 javascript 스크립트다.
- 이 스크립트가 백그라운드에서 실행되면서 서브에이전트에게 작업을 동시다발적으로 위임한다.


### 동작방식 예시)
프롬프트: `src/routes/ 아래 모든 API 엔드포인트에서 인증 체크 누락 감사해줘`

```javascript
const found = await agent('src/routes/ 아래 .ts 파일 전부 나열해줘', { schema: {...} })

const audits = await pipeline(found.files, file =>
  agent(`${file}에서 인증 체크 누락 감사해줘`, { label: file }),
)

return audits.filter(Boolean)
```

- 발견된 `.ts` 파일 수만큼 서브에이전트에게 위임한다.
- 파일이 300개면 300개의 서브에이전트가 병렬로 인증 체크를 감사한다.

워크플로우에 대한 자세한 내용은 [공식문서](https://code.claude.com/docs/ko/workflows)에서 확인.

워크플로우에 대한 내용은 별도 정리할 예정.

Claude가 ultracode를 활성화 하는 방법은 두 가지다.

### 1. 이번 작업만 한 번 켜기

- 세션 effort는 그대로 두고, 지금 시키는 작업 하나만 워크플로우로 처리하고 싶다면 프롬프트에 `ultracode` 키워드를 넣으면 된다.
- effort 레벨을 바꾸지 않고 그 작업에만 적용되기 때문에 평소엔 `high`나 `medium`으로 두고 큰 작업이 나올 때만 이 키워드로 워크플로우를 트리거하는 게 실용적이다.

```text
ultracode: src/routes/ 아래 모든 API 엔드포인트에서 인증 체크 누락 감사해줘
```

### 2. 세션 전체에 켜두기

- 켜두면 Claude가 세션의 모든 실질적인 작업마다 워크플로우가 필요한지를 스스로 판단한다.
- 요청 하나가 코드 파악용/수정용/검증용 워크플로우 여러 개로 쪼개지기도 해서, 매 요청마다 토큰 소모와 응답 시간이 늘어난다.

```bash
/effort ultracode          # 세션 중 전환, 또는 메뉴에서 선택
claude --effort ultracode  # 시작 시 (v2.1.203 이상 필요)
```

## 결론

작업의 유형에 따라 단순 모델만 결정하는 것보다 effort도 적절하게 설정하는 것도 토큰 절약, 결과 퀄리티에 중요한 영향을 미친다.

평소 코딩 작업은 `high` 기본값으로 충분하고, 복잡한 설계나 대규모 리팩터링처럼 판단이 많이 필요할 때만 `xhigh`/`max`로 올리는 식으로 쓰는 게 낫다.

`ultracode`는 별개로, 코드베이스 전체 감사나 대량 마이그레이션처럼 규모 자체가 큰 작업에서 사용하면 효율적으로 작업이 가능하다.

