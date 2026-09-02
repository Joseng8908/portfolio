---
# ─────────────────────────────────────────────────────────────────────────────
# 프로젝트 상세 페이지. 파일명이 그대로 URL이 됩니다:
#   content/portfolio/rdma-latency-harness.md  →  /portfolio/rdma-latency-harness/
# 새 프로젝트는 이 파일을 복사해서 쓰세요.
# ─────────────────────────────────────────────────────────────────────────────
title: "Etri 인턴기"

# 카드에 뜨는 한 줄 요약. 목록 페이지와 검색 인덱스가 이걸 씁니다.
summary: "Backend.AI를 기반으로 한 PD Disaggregated Serving"

# 상세 페이지에서 제목 바로 아래 뜨는 부제. 비워도 됩니다.
description: "ETRI"

# 카드 우측 상단. 생략 가능.
period: "[2026.07 – 2026.08]"

date: 2026-09-02
weight: 10          # 카드 정렬 순서. 작을수록 위. 10, 20, 30 … 으로 띄워두면 중간 삽입이 쉽습니다.
draft: false

# 카드 pill + 사이트 태그 인덱스(/tags/)에 함께 등록됩니다.
tags: ["Inference", "VLLM", "Backend.AI", "ETRI"]

ShowToc: true
TocOpen: false
---
## 개요
Backend.AI라는 AI 인프라 오케스트레이팅 플랫을 온프레미스 환경에 구축했다.
GPU prefill과 NPU decode를 분리해 70B모델을 서빙했다.

## 배경 / 문제
[왜 P/D 분리인가 — prefill은 연산집약(GPU 유리), decode는 메모리대역
 (NPU로 오프로드) → 이기종으로 나눠 각 단계를 최적 하드웨어에]

## 구성
[아키텍처 — A100×4(prefill, TP4) → KV Cache RDMA 전송 → NPU×4(decode)
 Mooncake RDMA/InfiniBand, Backend.AI 배포 관리]
[다이어그램 넣으면 좋음 — prefill→RDMA→decode 흐름]

## 핵심 작업
[내가 한 것 — KV 전송 경로 구성, host 네트워크 전환, 포트 배선,
 RDMA 실경로 검증(port_rcv_data)]

## 결과
[짧은 문답 워크로드 실서비스 가능 수준 goodput. P/D 분리+RDMA 정상 동작 검증]

## 배운 점 (선택)
[이기종 서빙의 실전 이슈 — 네트워크 격리, KV 전송 병목 등]

<!--
이미지 삽입:
  1) 이 파일을 페이지 번들로 바꾸면(권장) 이미지를 같은 폴더에 두고 상대경로로:
       content/portfolio/rdma-latency-harness/index.md
       content/portfolio/rdma-latency-harness/topology.png
       ![토폴로지](topology.png)
  2) 사이트 내부 링크는 반드시 relref — 절대경로는 프로덕션에서 404입니다:
       [관련 글]({{< relref "/posts/hello" >}})
-->
