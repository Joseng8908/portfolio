---
title: "(4일차) Golang으로 나만의 컨테이너 만들어보기"
date: 2026-03-23T13:32:44+09:00
lastmod: 2026-03-23T13:32:44+09:00
draft: false
tags: ["Go", "container", "linux", "docker", "networking"]
summary: "CLONE_NEWNET으로 네트워크를 격리하고, veth pair를 손으로 연결해 호스트와 통신시킨다."
ShowToc: true
TocOpen: false
---

## 서론

- 저번 일차 복습
    - 저번 일차에는 cgroup을 사용해서 container의 자원을 할당하고 제한하는 작업을 했다. 매우 많은 시행착오가 있었지만 어떻게 잘 해결했다.
- TODO
    - mount에 대해
        - 마운트는 특정 저장 장치나 파일 시스템을 OS의 특정 디렉토리에 연결하는 작업이다.
        - 컨테이너에서는 격리된 파일 시스템을 구축하기 위해 컨테이너 전용 파일들을 마운트하여 호스트의 정보(즉, 컨테이너를 돌리는 프로세스의 정보)를 컨테이너 내부로 들어가지 않게 한다.
    - cgroup에 대해
        - Control Group. Cgroup은 리눅스 커널의 기능으로, 프로세스 그룹의 자원 사용(CPU, 메모리, 디스크 I/O)등을 제한하고 관리한다.
        - 컨테이너에서는 특정 컨테이너에 자원 제한을 두거나, 특정 컨테이너에게 자원을 몰아주는 등 다양하게 사용한다.
        - 리눅스에서는 /sys/fs/cgroup경로에 폴더를 만들고 그 안의 파일에 값을 쓰는 방식으로 사용한다.
    - dd, tee에 대해
        - dd: 파일을 복사하거나 변환하는 명령어이다.
            - 용도: 빈 파일을 만들거나, 특정 크기의 디스크 이미지를 생성할 때 사용한다.
            - ex) dd if=/dev/zero of=rootfs.img bs=1M count 100(100MB의 빈 이미지 파일 생성하는 명령어)
        - tee: 표준 입력에서 읽은 내용을 화면에 출력하고, 파일에도 저장하는 명령어이다.
            - 용도: 컨테이너 설정 파일이나 cgroup 값을 수정할 때, 관리자 권한을 유지하면서 파일에 내용을 쓸 때 사용한다.
            - ex) echo "512M" | sudo tee /sys/fs/cgroup/memory/my-container/memory.limit_in_bytes
            - 단순히 리다이렉션 `>`을 사용하면 sudo 권한이 적용되지 않는 경우가 많기 때문에 tee를 사용한다.

## 본론

### Day 4 - Network 격리해보기

### 기본적인 파일 구조

```text
micro-docker/
    - cmd/
        - conatiner/
            - main.go # 프로그램 진입점
    - internal/
        - cgroups/
            - cgroups.go
        - container/
            - config.go # 컨테이너 관련 객체 선언
            - cotainer.go # 컨테이너 실행 명령어들, 그에 대한 환경설정 자동 세팅 메소드
            - run.go # 컨테이너 구동
```

### Code

#### 네트워크 격리하기.

**- 네트워크가 존재하지 않는 컨테이너를 띄우기**

```go
// container.go
...
cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS | // 호스트 명 격리
                    syscall.CLONE_NEWPID | // PID 격리(내부에서 PID 1번이 됨)
                    syscall.CLONE_NEWNS | // 마운트 격리
                    syscall.CLONE_NEWNET, // "네트워크 격리" 
        Chroot: "./rootfs", // 이게 컨테이너(격리된 프로세스의 입구)
    }
...
```

- 플래그에 CLONE_NEWNET을 추가해주면, 네트워크가 존재하지 않는 컨테이너를 만들 수 있다.

**- 격리된 컨테이너 띄우기**

```bash
sudo ./micro-container /bin/sh # 컴파일을 진행해서 그냥 실행파일로 명령어를 작성하겠다.
```

- 이 부분에서 PID를 확인해야 한다(나의 경우 1150이다).
- 앞으로 컨테이너 안의 명령어에는 /# 프롬프트를 사용하겠다.
- 없으면 호스트(부모 프로세스)를 의미하는 명령어이다.

**- 호스트(부모 프로세스)에서 가상 랜선 만들기**

```bash
# 터미널 2 (호스트용)
# 1. veth0(호스트쪽)와 veth1(컨테이너쪽) 쌍을 만듦
sudo ip link add veth0 type veth peer name veth1
```

**- 랜선 한 쪽을 컨테이너로 연결하기**

```bash
# 터미널 2 (호스트용)
# 2. veth1을 컨테이너(PID 1150) 네임스페이스로 이동
sudo ip link set veth1 netns 1150
```

- 이렇게 하면 컨테이너에서 결과가 아래처럼 생긴다

```text
/# ip addr
5: veth1@if6: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether ce:6e:37:5b:1e:82 brd ff:ff:ff:ff:ff:ff
```

**- 양쪽 연결을 유효하게 만들고, IP주소 부여하기**

- 호스트

```bash
sudo ip link set veth0 up
sudo ip addr add 172.17.0.1/24 dev veth0
```

- 컨테이너

```text
# 1. 인터페이스들 깨우기
/# ip link set lo up
/# ip link set veth1 up

# 2. 컨테이너 IP 부여 (172.17.0.2)
/# ip addr add 172.17.0.2/24 dev veth1
```

**- 통신 확인해보기**

- 컨테이너 쪽에서 호스트로 서버가 살아있는지 확인.

```text
/# ping 172.17.0.1
PING 172.17.0.1 (172.17.0.1): 56 data bytes
64 bytes from 172.17.0.1: seq=0 ttl=64 time=0.438 ms
64 bytes from 172.17.0.1: seq=1 ttl=64 time=0.125 ms
64 bytes from 172.17.0.1: seq=2 ttl=64 time=0.178 ms
64 bytes from 172.17.0.1: seq=3 ttl=64 time=0.131 ms
64 bytes from 172.17.0.1: seq=4 ttl=64 time=0.126 ms
^C
--- 172.17.0.1 ping statistics ---
```

### Observation(실행 결과)

- 최종 실행 결과를 보면 아마도 마지막에 통신 확인을 했던 ping명령어를 했을 때 나오는 로그일 것이다

```text
/ # ping 172.17.0.1
PING 172.17.0.1 (172.17.0.1): 56 data bytes
64 bytes from 172.17.0.1: seq=0 ttl=64 time=0.438 ms
64 bytes from 172.17.0.1: seq=1 ttl=64 time=0.125 ms
64 bytes from 172.17.0.1: seq=2 ttl=64 time=0.178 ms
64 bytes from 172.17.0.1: seq=3 ttl=64 time=0.131 ms
64 bytes from 172.17.0.1: seq=4 ttl=64 time=0.126 ms
^C
--- 172.17.0.1 ping statistics ---
```

### Why?(실행 결과 분석)

- 그래서 이게 무엇을 의미하나?
- 우리는 지금 논리적으로 완전히 분리된 두 개의 컴퓨터 사이에 가상 랜선을 연결하고, 통신 규약을 맞춘 것이다.
- 즉, 우리가 만든 컨테이너는 우주에 둥둥 떠있는 방 하나라고 볼 수 있었는데, 이 방과 우리의 집(호스트)과 연결하는 선을 깐 것이다.
- 그리고 이제 컨테이너 안에서 ping을 치게 되면, 그 둥둥 떠있는 방에서, 우리의 집과 **네트워크**로 즉, 실로 연결된 종이컵에 "거기 누구 있어요?" 하는 것마냥 생존 확인을 할 수 있다는 것이다.
- 이 실을 통해 이제 자원을 주고받고, 메시지를 주고받는 일을 할 수 있게 되는 것이다!

### Trouble Shooting

- 이번에는 딱히 없었다..아마 다음 주차때 이걸 코드로 구현하는 것에 많이 힘이 들 것 같은 기분이 든다.

## 결론

### Key Insights(핵심 정리)

- 네트워크 네임스페이스 격리(CLONE_NEWNET): 이 플래그를 사용함으로써 우리는 컨테이너와 호스트 사이의 모든 공유된 네트워크 자원을 고립시킬 수 있었다.
- veth pair: veth는 가상 장치이지만, 리눅스 커널 입장에서는 양 끝단이 연결된 실제 랜선과 똑같이 취급된다.
- 인터페이스 상태 관리(UP & LOWER_UP): 인터페이스를 up시키는 것만으로는 부족하다. 가상 랜선의 반대편 끝(peer)도 활성화되어야 LOWER_UP 상태가 되고, 데이터가 흐를 수 있는 통로가 완성된다.
- L3 통신을 위한 IP할당: 랜선이 연결되어도, 식별한 주소가 없으면 패킷을 보낼 수 없기에, 동일한 서브넷 대역(172.17.0.x/24)로 주소를 맞춰주는 작업이 필수이다.

### Next Step(다음 주제)

- 오늘 한 것을 이제 자동화 시켜서 컨테이너가 실행될 때 자동으로 호스트와 연결이 되는 코드를 짜볼 예정이다(netlink라이브러리 활용 예정).
- bridge 모드 구축?: 1:1연결 뿐만 아니라, 여러 컨테이너가 한 공유기에 접속하는 구조를 설계해볼 예정이다.

### TODO

- none
- 잡설: 이번 일차는 살짝 쉬어가는 날이었던 것 같다. 하지만, 다음 일차에 매우 빡센게 준비되어 있으니 긴장해야겠다....

### github repo 주소

[https://github.com/Joseng8908/go_programming](https://github.com/Joseng8908/go_programming)
