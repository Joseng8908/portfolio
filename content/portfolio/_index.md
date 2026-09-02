---
title: "ETRI 인턴기"
summary: "Backend.AI 기반 이기종 가속기(GPU + NPU) P/D Disaggregated Serving"
description: "ETRI 네트워크AI컴퓨팅 연구실 하계 인턴"
period: "2026.07 – 2026.08"
date: 2026-09-02
weight: 10
draft: false
tags: ["AI Inference", "vLLM", "Backend.AI", "RDMA", "ETRI"]
ShowToc: true
TocOpen: false
---

## 개요

GPU(prefill)랑 NPU(decode)를 분리한 이기종 파이프라인으로 Llama-3.3-70B를 서빙하고, goodput이 안 나오는 병목이 연산(TPOT)이 아니라 요청 입장 경로(TTFT)에 있다는 걸 규명했다.

Backend.AI라는 AI 인프라 오케스트레이션 플랫폼을 온프레미스 환경에 구축하고, 그 위에서 P/D 분리 서빙과 다른 일반 모델들을 돌리고 운영하였다. 플랫폼이 원래 지원 안 하던 NPU를 직접 편입하는 것부터 시작해서, 컨테이너 기반의 KV Cache RDMA 전송 경로 구성, 성능 벤치마크, 병목 분석까지 전 과정을 담당했다.

## 배경 — 왜 P/D를 분리하나

LLM 추론은 두 단계의 성격이 완전히 다르다. 입력 전체를 한 번에 처리하는 **prefill**은 연산 집약적이라 연산 성능 높은 GPU가 유리하고, 토큰을 하나씩 뽑아내는 **decode**는 메모리 대역폭에 지배되고 연산 부하는 상대적으로 낮다.

이 둘을 같은 장치에서 처리하면 서로 특성이 달라서 자원을 비효율적으로 쓴다. 그래서 각 단계를 제일 잘 맞는 하드웨어에 나눠 붙이는 게 P/D 분리(Prefill/Decode Disaggregation)다. 이 프로젝트에선 prefill을 GPU(A100)에, decode를 NPU(Tenstorrent)에 오프로드해서 이기종 구성이 실제로 되는지를 검증하는 게 목표였다.

## 구성

```
[입력] → prefill (A100×4, TP4) → KV Cache → [RDMA] → decode (NPU×4) → [출력]
                                             Mooncake / InfiniBand
```

- **prefill 노드**: A100 80GB × 4, Tensor Parallelism 4로 vLLM 구동
- **decode 노드**: Tenstorrent NPU × 4, tt-metal 런타임 위에서 vLLM
- **KV Cache 전송**: prefill이 만든 KV Cache를 Mooncake RDMA(InfiniBand)로
  decode 노드에 넘김 — P/D 분리의 핵심 데이터 경로
- **오케스트레이션**: 전 컴포넌트를 Backend.AI 배포 단위로 관리하고, proxy가
  prefill·decode를 중계

<!-- 아키텍처 그림 넣을 자리. 이미지를 이 폴더(etri/)에 두고 아래처럼:
![아키텍처](topology.png)
-->

## 핵심 작업

- **Backend.AI 플랫폼 구축**: 여러개의 서버와 이기종 가속기를 기반으로 Backend.AI를 설치, 확장, 운영하며 실제 사내 LLM서비스를 다양한 모델(K-Exaone, GLM, Llama, A.X 등등)을 사용하여 운영하였다.
- **플랫폼에 신규 NPU 편입**: n300 전제로 짜여 있던 가속기 플러그인을 신규 카드도 인식하도록 패치하고, 자원 슬롯 등록. 5건 넘는 플랫폼 이슈를 하나씩 규명·수정해서 NPU 4장을 정상 인식시켰다.
- **KV 전송 경로 구성**: prefill(producer) → decode(consumer) 간 RDMA 기반 KV Cache 전송을 컨테이너 하에 구성하고, 요청 처리 전 구간이 실제로 도는지까지 검증했다.
- **host 네트워크 전환**: bridge 모드에선 컨테이너 내부 주소가 광고돼서 KV전송이 실패했다. host 네트워크로 바꾸고 플랫폼의 포트 처리 로직을 패치해서 해결했다.

## 결과

- GPU만으로 안 채우고 **이기종 가속기(GPU + NPU)로 LLM 서빙이 성립**한다는 것을 실제로 보여줬다.
- 2모델 × 4워크로드 × 8요청률 = **64측정점** goodput 벤치마크를 돌리고, goodput 제약이 TPOT가 아니라 **TTFT(요청 입장 경로) 병목**에서 온다는 걸 규명했다.
- 컨테이너로 돌릴 때 생기는 성능 오버헤드(TPOT, TTFT의 지연)의 원인을 `schedstat` 기반으로 실측해서 짚었다.
- 짧은 문답 워크로드에선 실서비스 가능한 수준의 goodput을 확인했다.

<!-- 벤치 그래프 넣을 자리 (TPOT vs TTFT 대조 등):
![TPOT vs TTFT](tpot-vs-ttft.png)
-->

## 배운 점

- **네트워크 격리가 분산 추론의 숨은 병목이다.** 컨테이너 네트워크 모드 하나가 RDMA KV 전송의 성패를 갈랐다. "컨테이너 안에서 보는 자기 주소 ≠ 밖에서 도달 가능한 주소"라는 NAT 격리의 함정을 직접 밟으면서 이해했다.
- 이기종 서빙의 진짜 난점은 개별 가속기 성능이 아니라 그것들을 **잇는 경로**(KV 전송, 네트워크, 스케줄링)에 있다는 걸 체감했다.

## Future Work
- 현재 NPU가 컨테이너 하에서 돌아갈 때 CPU를 사용하는 것은 관찰했지만, 그 이유를 규명하지 못하였다.
- GLM 5.2는 RTX Pro 6000ada를 사용했는데 아직 sm120을 vLLM쪽에서 지원하지 못한다.

<!--
이미지 넣는 법:
  이 파일이 이미 page bundle(etri/index.md)이니까, 이미지를 같은 폴더(etri/)에
  두고 상대경로로 참조하면 된다:
      content/portfolio/etri/topology.png
      ![아키텍처](topology.png)

  사이트 내부 링크는 반드시 relref (절대경로는 배포하면 404):
      [관련 글]({{< relref "/posts/hello" >}})
-->