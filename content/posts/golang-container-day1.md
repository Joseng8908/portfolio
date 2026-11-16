---
title: "(1일차) Golang으로 나만의 컨테이너 만들어보기"
date: 2026-03-10T18:09:17+09:00
lastmod: 2026-03-10T18:09:17+09:00
draft: false
tags: ["Go", "container", "linux", "docker"]
summary: "Go로 나만의 docker 만들기 1일차 — 기본 구조를 짜고 Namespace로 격리된 프로세스를 띄운다."
ShowToc: true
TocOpen: false
---

## 서론

- golang을 다양한 책으로 공부하고 있지만, 주로 짜게 되는 코드들은 gin을 이용한 백엔드 서버가 많았다. 그렇기에 내가 원하는 시스템 프로그래밍에 관한 구조와 코드를 짜 본 적이 없어, 공부해놓은 linux, unix, os 등의 개념을 직접 사용하는 느낌이 들지 않았다. 때문에 docker라는 백엔드 배포 환경에서 사용되는 프로그램을 shadowing하며 나만의 docker를 만들어보기로 했다.
- 개발 환경은 linux - ubuntu 24.04LTS 를 사용하고 있으며, wsl을 통해 windows 환경에서 lunarvim(neovim의 기능 추가 버전)을 사용하여 개발하고 있다. 이 프로젝트에 사용하는 go version은 go 1.22.2를 사용 중이다.

## 본론

### Day 1 - 기본적인 구조 짜고 프로세스 돌리기

### 기본적인 파일 구조 짜기

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

컨테이너 하나 띄우고 관리하는 구조체 정의하기

**- 구조체 정의**  
컨테이너의 상태를 관리하고, 외부에서 제어(종료)할 수 있도록 Context를 포함했다.

```go
type Container struct {
    ID     string // Continer id
    Cmd    *exec.Cmd // 실행할 명령어 
    Status string // Created, Running, Stopped 등 상태 정의
    Ctx context.Context // 상태 보내는 context
    Cancel context.CancelFunc // 긴급 정지 필드
}
```

**- 새로운 컨테이너 만드는 로직**  
SysProcAttr를 사용하여 리눅스 커널에게 격리 환경(Namespace)을 요청한다.

```go
func NewContainer(id string, command string, args ...string) *Container {
    // cmd 객체 만들고, 명령어 넣기 (명령어는 command line으로 받을 예정)
    cmd := exec.Command(command, args...)

    // 격리 설정
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS | // 호스트 명 격리
                    syscall.CLONE_NEWPID | // PID 격리(container내부에서는 자신이 PID 1번이 됨)
                    syscall.CLONE_NEWNS, // 마운트 격리
    }

    // container에 대한 정보 객체 반환
    return &Container{
        ID: id,
        Cmd: cmd,
        Status: "Created",
        Ctx: ctx,
        Cancel: cancel,
    }
}
```

**- 컨테이너 프로세스 실행 로직**

```go
func (c *Container) Start() error {
    // 만들어 두었던 cmd객체에 있는 명령어대로 실행
    if err := c.Cmd.Start(); err != nil {
        return err
    }
    // 확인용 로그
    fmt.Printf("컨테이너 ID: %s, PID: %d 로 실행됨\n", c.ID, c.Cmd.Process.Pid)
    return nil
}
```

**- main.go, 프로그램 진입점**

```go
func main() {
    if len(os.Args) < 2 {
        fmt.Println("사용법: go run main.go <command>")
        return
    }
    // command line 명령어 받기
    cmd := os.Args[1] // 명령어 
    args := os.Args[2:] // 인자

    fmt.Printf("새로운 컨테이너 생성중,,,%s\n", cmd)

    // 컨테이너 생성
    c := container.NewContainer("1", cmd, args...)
    // 컨테이너 실행
    if err := c.Start(); err != nil {
        fmt.Printf("컨테이너 실행 실패: %v\n", err)
        return
    }

    // 프로세스 대기
    err := c.Cmd.Wait()
    if err != nil {
        fmt.Printf("컨테이너 에러: %v\n", err)
    } else {
        fmt.Println("컨테이너 종료")
    }
}
```

### Observation(실행 결과)

```text
# bin/sh, 즉 쉘 실행 파일이므로 super user권한으로 실행시켜야 한다
sudo go run main.go bin/sh
새로운 컨테이너 생성중,,,/bin/sh
컨테이너 ID: 1, PID 3394로 실행됨
컨테이너 종료
```

- 로그를 보면 /bin/sh를 실행하는 컨테이너가 실행되고, 그 내부 컨테이너 ID(가짜 PID)와 외부에서 보이는 실제 그 컨테이너의 PID가 나온 후, 바로 컨테이너가 종료되는 것을 볼 수 있다
- 하지만 /bin/sh를 실행하면 shell이 실행되면서 명령어를 받을 수 있는 shell prompt가 나와야 정상인데 바로 프로그램이 꺼진다. 왜 그럴까?

### Why?(실행 결과 분석)

- 실행 결과를 보면 /bin/sh가 실행되자마자 바로 종료된다. 이는 입출력(I/O)이 호스트 터미널과 연결되지 않았기 때문이다.
- /bin/sh는 사용자로부터 명령어를 입력받아야 하는 대화형 프로세스인데, 연결된 표준 입력(Stdin)이 없으니 "할 일이 없다"고 판단하여 스스로 프로세스를 종료(EOF)한 것이다.

### Key Insights(핵심 정리)

- 오늘 구현의 핵심은 **SysProcAttr**이다. 이는 리눅스 커널에게 프로세스를 생성할 때 **"이 프로세스는 호스트와 격리된 방(Namespace) 안에서 살아야 해"**라고 명시하는 설계도다. 우리는 이 설계도를 바탕으로 격리된 환경에서 프로세스를 띄우는 데 성공했다.

### Mental Model(비유)

- SysProcAttr: 리눅스 커널에게 전달하는 '격리 설계도'. 이를 통해 프로세스가 태어날 때부터 격리된 Namespace를 할당받게 한다.
- 컨테이너: 격리 설계도를 기반으로 특정 명령어를 실행하는 '밀폐된 방'.

### Next Step(다음 주제)

1. 입출력(I/O) 연결: 컨테이너의 표준 입출력을 호스트 터미널과 연결하여 대화형 쉘 구현하기.
2. 파일시스템 격리: chroot를 사용하여 컨테이너가 호스트의 루트 파일시스템과 별개로 살게 만들기.

### github repo 주소

[https://github.com/Joseng8908/go_programming](https://github.com/Joseng8908/go_programming)
