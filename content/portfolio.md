---
title: "Portfolio"
url: "/portfolio/"
layout: "portfolio"
summary: "Systems / infrastructure engineering portfolio — selected projects."
ShowToc: false
ShowReadingTime: false
ShowBreadCrumbs: false
hidemeta: true

# ─────────────────────────────────────────────────────────────────────────────
# Header block
# ─────────────────────────────────────────────────────────────────────────────
# Path is relative to assets/. Resized to 92px (2x for retina) by the template.
avatar: "images/gopher.png"
avatarAlt: "Gopher avatar"

name: "[이름]"
tagline: "[한 줄 소개 — 예: 커널·네트워크 계층에서 성능을 측정하고 재현하는 인프라 엔지니어]"

# 기간 / 소속. 각 항목이 가운데점(·)으로 구분되어 한 줄로 붙습니다.
affiliations:
  - "[ETRI 하계 인턴]"
  - "[2026. 여름]"
  - "[서울과학기술대학교]"

contacts:
  - label: "GitHub"
    url: "https://github.com/Joseng8908"
  - label: "Blog"
    url: "/posts/"
  - label: "Email"
    url: "mailto:joseng@g.seoultech.ac.kr"

# ─────────────────────────────────────────────────────────────────────────────
# Projects — 5 blocks.
#   title   : 제목
#   meta    : 기간 / 역할 / 규모 (선택)
#   summary : 한 줄 요약
#   points  : 핵심 기여 2~3개. 마크다운 인라인 사용 가능 (`code`, **bold**, [link](url))
#   tags    : 기술 태그 (pill로 렌더)
#
# 불릿은 "무엇을 했다"보다 "무엇이 얼마나 바뀌었다"로 쓰는 편이 30초 스캔에
# 유리합니다. 숫자가 있으면 숫자를 먼저.
# ─────────────────────────────────────────────────────────────────────────────
projectsTitle: "Selected Projects"
projects:
  - title: "[프로젝트 1 제목]"
    meta: "[2026.07 – 2026.08 · 개인]"
    summary: "[한 줄 요약 — 무엇을 만들었고 왜 만들었는지]"
    points:
      - "[핵심 기여 1 — 측정 가능한 결과를 앞에 두기. 예: p99 latency 41ms → 12ms]"
      - "[핵심 기여 2 — 어떤 기술적 판단을 했는지]"
      - "[핵심 기여 3 — 선택]"
    tags: ["Go", "Linux", "eBPF"]

  - title: "[프로젝트 2 제목]"
    meta: "[기간 · 역할]"
    summary: "[한 줄 요약]"
    points:
      - "[핵심 기여 1]"
      - "[핵심 기여 2]"
    tags: ["Kubernetes", "Helm", "ArgoCD"]

  - title: "[프로젝트 3 제목]"
    meta: "[기간 · 역할]"
    summary: "[한 줄 요약]"
    points:
      - "[핵심 기여 1]"
      - "[핵심 기여 2]"
      - "[핵심 기여 3]"
    tags: ["RDMA", "NUMA", "perf"]

  - title: "[프로젝트 4 제목]"
    meta: "[기간 · 역할]"
    summary: "[한 줄 요약]"
    points:
      - "[핵심 기여 1]"
      - "[핵심 기여 2]"
    tags: ["Terraform", "GCP", "GitHub Actions"]

  - title: "[프로젝트 5 제목]"
    meta: "[기간 · 역할]"
    summary: "[한 줄 요약]"
    points:
      - "[핵심 기여 1]"
      - "[핵심 기여 2]"
    tags: ["PostgreSQL", "Redis", "Prometheus"]

# ─────────────────────────────────────────────────────────────────────────────
# Technical stack — 하단 태그 모음. group 단위로 줄이 나뉩니다.
# ─────────────────────────────────────────────────────────────────────────────
stackTitle: "Technical Stack"
stack:
  - group: "Languages"
    items: ["Go", "Java / Spring Boot", "Bash", "SQL"]
  - group: "Systems"
    items: ["Linux", "eBPF", "perf", "NUMA", "RDMA"]
  - group: "Orchestration"
    items: ["Kubernetes", "Docker", "Helm", "ArgoCD"]
  - group: "Infrastructure"
    items: ["Terraform", "Ansible", "GCP", "Azure"]
  - group: "Data"
    items: ["PostgreSQL", "Redis"]
  - group: "Observability"
    items: ["Prometheus", "Grafana"]
---

[선택 — 이 영역은 일반 마크다운입니다. 헤더 바로 아래, 프로젝트 목록 위에
들어갑니다. 지원 분야나 관심사를 2~3문장으로 적거나, 비워두면 아예
렌더링되지 않습니다.]
