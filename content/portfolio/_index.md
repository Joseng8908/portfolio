---
title: "Portfolio"
summary: "Systems / infrastructure engineering portfolio — selected projects."
ShowToc: false
ShowReadingTime: false
ShowBreadCrumbs: false
hidemeta: true

# ─────────────────────────────────────────────────────────────────────────────
# Header block — 왼쪽 아바타, 오른쪽에 이름 · 한 줄 소개 · 연락처 · 관심사
# ─────────────────────────────────────────────────────────────────────────────
# Path is relative to assets/. Resized to 132px (2x for retina) by the template.
avatar: "images/gopher.png"
avatarAlt: "Gopher avatar"

name: "조상윤"
tagline: "Infra / system engineer. Go, Linux internals, Kubernetes."

contacts:
  - label: "GitHub"
    url: "https://github.com/Joseng8908"
  - label: "Blog"
    url: "/posts/"
  - label: "joseng@g.seoultech.ac.kr"
    url: "mailto:joseng@g.seoultech.ac.kr"

interestsTitle: "Interests"
interests:
  - "금융권 IT"
  - "LLM 서빙 인프라"
  - "컨테이너 런타임"
  - "Kubernetes 스케줄링"
  - "RDMA / 네트워크"
  - "Linux 커널"

# ─────────────────────────────────────────────────────────────────────────────
# CV rows — period / org / role / summary. url은 선택(내부 경로 또는 http).
# 섹션: experience, activities, awards, education (비우면 섹션이 사라짐)
# ─────────────────────────────────────────────────────────────────────────────
experienceTitle: "Experience"
experience:
  - period: "2026.07 – 2026.08"
    org: "ETRI 네트워크AI컴퓨팅연구실"
    role: "연구연수생 (하계 인턴)"
    summary: "Backend.AI 기반 GPU/NPU LLM 서빙 인프라 구축·운영, 이기종 P/D 분리 서빙의 컨테이너화"
    url: "/portfolio/etri/"

activitiesTitle: "Activities"
activities:
  - period: "2026"
    org: "멋쟁이사자처럼 대학 14기 · 서울과학기술대학교"
    role: "백엔드 파트 운영진"
  - period: "2026.04 – 2026.06"
    org: "오픈소스 컨트리뷰션 아카데미 (OSSCA, 체험형)"
    role: "수료"
    summary: "Git 활용 및 Cilium 프로젝트 참여"

awardsTitle: "Awards"
awards:
  - period: "2026.08"
    org: "멋쟁이사자처럼 대학 14기 중앙 해커톤 〈애니멀리그〉"
    role: "우수상"
    summary: "약 400팀 중 3등"

educationTitle: "Education"
education:
  - period: "2022.03 – 재학 중"
    org: "서울과학기술대학교"
    role: "컴퓨터공학과"

projectsTitle: "Selected Projects"

# ─────────────────────────────────────────────────────────────────────────────
# Technical stack — 하단 태그 모음. group 단위로 줄이 나뉩니다.
# ─────────────────────────────────────────────────────────────────────────────
stackTitle: "Technical Stack"
stack:
  - group: "Languages"
    items: ["Go (primary)", "C", "Python (learning)"]
  - group: "Orchestration"
    items: ["Kubernetes"]
  - group: "Infrastructure"
    items: ["Docker Compose", "Backend.AI"]
  - group: "Observability"
    items: ["Prometheus", "Grafana"]
---
