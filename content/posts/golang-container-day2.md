---
title: "(2일차) Golang으로 나만의 컨테이너 만들어보기"
date: 2026-03-12T21:12:55+09:00
lastmod: 2026-03-12T21:12:55+09:00
draft: false
tags: ["Go", "container", "linux", "docker"]
summary: "컨테이너의 표준 입출력을 터미널에 연결하고, chroot로 루트 파일시스템을 격리한다."
ShowToc: true
TocOpen: false
---

## 서론

- 저번 일차 복습
    - 저번 일차에는 기본적인 파일 구조를 짰다. 그 후, 기본적인 격리된 프로세스를 만드는 container.go 로직과, 로직들을 연결하여 프로그램 진입점 역할을 하는 main.go를 짰다
    - 컨테이너에 대한 기본적인 정보를 정의하는 구조체를 만들고, SysProcAttr 라는 객체의 설정값을 부여하여 호스트, PID, 마운트를 격리시켰다.
    - command line에서 명령어를 받고, 해당 명령어에 실행할 프로그램 파일 경로를 인자로 넣어, 격리된 프로세스에 해당 프로그램을 띄우는 실험을 하였다. 이 도중, /bin/sh 이라는 shell을 실행하는 명령어를 인자로 넣었는데, input경로를 격리된 프로세스가 찾지 못해, 그냥 바로 실행을 종료시키는 문제가 발생했다.
    - 또한, root와 container는 같은 파일시스템을 공유하지 않고, 필요할 때 가져오거나, 특정 파일만 공유하는 파일 추출 및 볼륨 마운트 기능을 원래 사용하는데, 지금 내가 만든 container는 모든 파일시스템을 공유하기 때문에 이 부분을 수정할 예정이다.

## 본론

### Day 2 - 컨테이너 입출력 경로 조정하고, 컨테이너와 루트 파일시스템 격리하기

### 기본적인 파일 구조

```text
micro-docker/
    - main.go //프로그램 진입점
    - container/
        - container.go // 새로운 컨테이너 생성 로직 및 실행 로직
        - run.go // Namespace 설정 및 프로세스 실행 로직
    - monitor/
        - monitor.go // 프로세스 상태 확인 및 자원 정리
```

### Code

#### 1. 컨테이너 입출력 경로 조정하기.

**- 만든 cmd객체의 표준 입출력을 터미널과 연결하는 로직**

```go
// container.go 에서 
... 
cmd := exec.Command(command, args...) // 만들어진 cmd객체의 표준 입출력을, 터미널과 연결 
cmd.Stdin = os.Stdin 
cmd.Stdout = os.Stdout 
cmd.Stderr = os.Stderr ctx, cancel := context.WithCancel(context.Backgroud()) 
...
```

### Observation(실행 결과)

```text
sangyun8908@LAPTOP-OVR25738:~/go_programming/micro_container$ sudo go run main.go /bin/sh
[sudo] password for sangyun8908:
새로운 컨테이너 생성중,,,/bin/sh
컨테이너 ID: 1, PID: 4823 로 실행됨
# ls
alpine-minirootfs-3.19.1-x86_64.tar.gz  go.mod   monitor
container                               main.go  rootfs
#
```

- 터미널에서 실행해봤더니 컨테이너가 바로 꺼지지 않고 # 프롬프트가 뜨면서 /bin/sh 프로세스가 실행되고 있다는 것을 볼 수 있다.

### Why?(실행 결과 분석)

- 앞의 실행 결과를 보면 컨테이너가 잘 실행되고 있다. 하지만 지금 큰 문제점이 있다, 바로 ls(list) 명령어를 사용했을 때, 지금 현재 디렉토리에 있는 파일의 목록이 뜬다는 것이다. 이것이 문제가 되는 이유는 container는 격리되어 있는 환경이기 때문에 파일 시스템을 wsl에 있는 파일들과 연결되어 있으면 안 되는데, 지금 연결되어 있다는 것이다. 즉, "격리"가 되어있지 않다는 뜻이다. 이제 이를 해결해보자.

#### 2. 컨테이너와 루트 파일시스템 격리하기

**- 필요한 파일 설치하기**

- 컨테이너 위에서 다양한 프로그램을 실행하기 위해서는 그 실행 파일들과 다양한 의존성, 링크 파일들이 필요하다. 그렇기 때문에 직접 기존의 linux파일들을 복사해서 가져오거나, Alpine Rootfs? 라는 배포되어 있는 버전을 다운받아야 한다. 나는 이 두 방법 중 Alpine Rootfs를 다운로드 했다.

```bash
# Alpine Rootfs 다운로드 
~/go_programminng/micro_container/$ wget https://dl-cdn.alpinelinux.org/alpine/v3.19/releases/x86_64/alpine-minirootfs-3.19.1-x86_64.tar.gz 
# 압축 파일을 풀고 파일들을 담아놓을 디렉토리 생성 
~/go_programminng/micro_container/$ mkdir rootfs 
# 압축 파일 풀고 rootfs에 넣기 
~/go_programminng/micro_container/$ sudo tar -xf alpine-minirootfs-3.19.1-x86_64.tar.gz -C ./rootfs
```

- 이렇게 하면 /rootfs/ 에 이러한 파일들이 생긴다

```text
sangyun8908@LAPTOP-OVR25738:~/go_programming/micro_container/rootfs$ ls bin etc lib mnt proc run srv tmp var dev home media opt root sbin sys usr
```

**- 컨테이너와 루트 파일시스템 격리하는 로직**

- 이제 앞서 만든 rootfs/ 디렉토리를 새로 만들어지는 컨테이너의 홈 디렉토리로 지정을 한다.

```go
container.go 
... 
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS | // 호스트 명 격리
                    syscall.CLONE_NEWPID | // PID 격리(내부에서 PID 1번이 됨)
                    syscall.CLONE_NEWNS, // 마운트 격리
        Chroot: "./rootfs", // 이게 컨테이너(격리된 프로세스의 입구)
    }
    cmd.Dir = "/" // 프로세스가 시작될 위치를 컨테이너 내부의 루트로 고정하기

...
```

### Observation(실행 결과)

```text
sangyun8908@LAPTOP-OVR25738:~/go_programming/micro_container$ sudo go run main.go /bin/sh
새로운 컨테이너 생성중,,,/bin/sh
컨테이너 ID: 1, PID: 5406 로 실행됨
/ # ls
bin    etc    lib    mnt    proc   run    srv    tmp    var
dev    home   media  opt    root   sbin   sys    usr
```

- 실행 결과를 보면 컨테이너가 /bin/sh 프로세스를 실행하는 것을 볼 수 있다(# 쉘 프롬프트가 뜬다). 또한, 내부에서 ls를 했을 때 파일시스템이 wsl과 공유되지 않고, 자신 고유의 파일 시스템을 반환하는 것을 볼 수 있다. 또한, cd ..을 했을 때(자신의 상위 폴더로 올라가는 명령어) 자신의 root 프로세스의 파일 시스템에 접근하지 않는다! 이 점이 매우매우 좋았다.

### Why?(실행 결과 분석)

- 그런데 솔직히 말하면 왜 자신의 root 프로세스의 파일 시스템에 접근하지 않는지 그 원리를 모른다.

### Trouble Shooting

- 또한, 실행 중 오류가 났었다. 근데 이게 좀 애매한 부분이 있다.
- 오류를 소개해 보겠다.

**1. 첫번째 오류**

```text
sangyun8908@LAPTOP-OVR25738:~/go_programming/micro_container$ ls -l ./rootfs/bin/sh lrwxrwxrwx 1 root root 12 Jan 27 2024 ./rootfs/bin/sh -> /bin/busybox
```

- 바로 링크가 다른 곳으로 되어있는 문제였다. 즉, rootfs/bin/sh를 실행해야 하는데, ./rootfs/bin/sh 가 /bin/busybox를 실행하고 있었다는 것이다. 다시 말하자면, 우리가 만든 컨테이너가 그것의 안에 있는 파일이 아닌 root파일시스템에 있는 파일을 찾고 있었다는 것이다.

```bash
cd rootfs/bin
sudo rm sh
sudo ln -s busybox sh
```

- 그래서 rootfs/bin 파일의 sh을 지우고, rootfs/bin/busybox로 이어지는 심볼릭 링크인 sh을 다시 만들어 주었다.

**2. 두번째 오류: 하지만, 링크를 수정해도 경로를 못 찾는다는 오류가 떴다.**

- 그런데....잘 찾아보니 container.go 안의 파일에 `rootfs` 가 아닌 `roofs`로 오타가 되어있는 것을 찾았다.....

## 결론

### Key Insights(핵심 정리)

- RootFS (파일 시스템): chroot 시스템 콜을 통해 프로세스가 인지하는 시스템의 루트를 우리가 준비한 rootfs 디렉토리로 제한하여, 호스트 파일 시스템으로부터 완벽히 고립시켰다.
- 디버깅의 교훈: 작은 경로 오타(rootfs vs roofs) 하나가 시스템 전체의 실행 실패를 가져올 수 있음을 깨달았다. 실제 리눅스 시스템은 경로 해석(Path Resolution)에 극도로 엄격하며, 이는 보안과 격리 측면에서 매우 중요한 원칙임을 배웠다.

### Mental Model(비유)

- 오늘은 skip

### Next Step(다음 주제)

- Cgroups를 이용한 자원통제

### TODO

- 어떻게 cd ..를 해도 상위 폴더로 이동하지 않는건지...로직이 궁금하다.
- 심볼릭 링크에 대해 다시 공부해야 할 것 같다.

### github repo 주소

[https://github.com/Joseng8908/go_programming](https://github.com/Joseng8908/go_programming)
