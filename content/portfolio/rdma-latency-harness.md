---
# ─────────────────────────────────────────────────────────────────────────────
# 프로젝트 상세 페이지. 파일명이 그대로 URL이 됩니다:
#   content/portfolio/rdma-latency-harness.md  →  /portfolio/rdma-latency-harness/
# 새 프로젝트는 이 파일을 복사해서 쓰세요.
# ─────────────────────────────────────────────────────────────────────────────
title: "[프로젝트 1 제목]"

# 카드에 뜨는 한 줄 요약. 목록 페이지와 검색 인덱스가 이걸 씁니다.
summary: "[한 줄 요약 — 무엇을 만들었고 왜 만들었는지]"

# 상세 페이지에서 제목 바로 아래 뜨는 부제. 비워도 됩니다.
description: "[상세 페이지 부제 — summary와 같아도 무방]"

# 카드 우측 상단. 생략 가능.
period: "[2026.07 – 2026.08]"

date: 2026-09-02
weight: 10          # 카드 정렬 순서. 작을수록 위. 10, 20, 30 … 으로 띄워두면 중간 삽입이 쉽습니다.
draft: false

# 카드 pill + 사이트 태그 인덱스(/tags/)에 함께 등록됩니다.
tags: ["RDMA", "NUMA", "Go", "perf"]

ShowToc: true
TocOpen: false
---

[여기부터 본문입니다. 일반 블로그 글과 완전히 동일하게 렌더링됩니다 —
같은 본문 폭, 같은 폰트, 같은 코드 하이라이팅, 목차 자동 생성.
A4 1~2장 분량으로 자유롭게 쓰시면 됩니다.]

## 배경

[왜 이걸 만들었는지. 어떤 문제가 있었고, 기존 방법으로는 왜 부족했는지.]

## 접근

[어떻게 풀었는지. 구조적 판단과 그 근거.]

### 측정 방법

[무엇을, 어떤 조건에서, 어떻게 쟀는지. 재현 가능하게.]

```go
// [코드 블록은 라인 넘버 + 복사 버튼이 자동으로 붙습니다.]
func measure(ctx context.Context, cfg Config) (*Result, error) {
	if err := pinToNode(cfg.Node); err != nil {
		return nil, fmt.Errorf("measure: %w", err)
	}
	// ...
	return &Result{P99: p99, P50: p50}, nil
}
```

```bash
# [셸 블록도 동일합니다.]
numactl --cpunodebind=0 --membind=0 ./harness --duration 60s
```

## 결과

[숫자를 앞에 두기. 표가 읽기 편합니다.]

| 조건 | p50 | p99 | 비고 |
|---|---:|---:|---|
| [baseline] | [—] | [—] | [—] |
| [tuned] | [—] | [—] | [—] |

## 배운 것

[예상과 달랐던 지점. 다음에 다시 한다면 무엇을 바꿀지.]

<!--
이미지 삽입:
  1) 이 파일을 페이지 번들로 바꾸면(권장) 이미지를 같은 폴더에 두고 상대경로로:
       content/portfolio/rdma-latency-harness/index.md
       content/portfolio/rdma-latency-harness/topology.png
       ![토폴로지](topology.png)
  2) 사이트 내부 링크는 반드시 relref — 절대경로는 프로덕션에서 404입니다:
       [관련 글]({{< relref "/posts/hello" >}})
-->
