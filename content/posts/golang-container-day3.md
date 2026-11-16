---
title: "(3일차) Golang으로 나만의 컨테이너 만들어보기"
date: 2026-03-17T16:43:19+09:00
lastmod: 2026-03-17T16:43:19+09:00
draft: false
tags: ["Go", "container", "linux", "docker", "cgroup"]
summary: "cgroup v2로 컨테이너의 메모리를 제한하고, swap 설정과 정리 로직까지 다룬다."
ShowToc: true
TocOpen: false
---

## 서론

- 저번 일차 복습
    - 저번 일차에는 컨테이너 입출력 경로를 조정하여 컨테이너가 바로 꺼지는 문제를 해결했다
    - 그리고 컨테이너와 루트의 파일시스템을 격리했다
- TODO
    - 컨테이너 격리를 cmd.Dir와 SysProcAttr에 Chroot 항목으로 지정된 디렉토리를 이용해서 했더니 cd ..를 해도 루트 파일시스템으로 안 들어가는 이유(즉, 격리가 잘 되는 이유와, 그 내부 로직)
        - 왜 `cd ..` 를 하면 상위 디렉토리로 가지 않을까? 그 이유는 os커널의 특성 때문이다. 커널은 자신이 현재 루트 디렉토리(/)에 있으면 당연하게 그 위는 없다고 생각하게 된다. 즉, 우리는 격리를 시켜서 `rootfs`를 프로세스의 루트 디렉토리로 지정했기 때문에, 그 프로세스는(커널도 어떻게 보면 프로세스니깐) 자신이 루트 디렉토리에 있고, 그 상위 디렉토리가 없다고 생각하기 때문에 상위 디렉토리로의 이동이 이루어지지 않는다는 것이다
        - 추가: 제미나이에 따르면 현대의 컨테이너 기술은 PID, 네트워크 자원까지 완벽히 격리하지 못하는 Chroot말고, Pivot_root와 Namespaces를 조합해서 사용한다고 한다.
    - 심볼릭 링크에 대하여
        - 심볼릭 링크는 원본 파일의 위치를 담고 있는 특수한 파일이다! 원본 파일이 삭제되거나 이동하면, 링크가 깨진다. 특이하게도 서로 다른 파티션이나 디스크 간에도 연결이 가능하다. 이 부분에서 절대 경로와 상대 경로의 위험성이 느껴진다... 이거 항상 잘 구분하고 사용해야겠다고 생각했다.
    - 추가로 궁금한 점
        - 심볼릭 링크가 원본파일을 가리킨다면,,, 원래 sh은 busybox라는 파일을 가리키고 있던 것인가?
        - 맞다고 한다! 그럼 busybox가 뭘까? 바로 리눅스 시스템에 필요한 수많은 명령어들을 하나로 합쳐놓은 실행파일이라 한다...어쩐지 sh파일에 내가 생각한 로직이 없던 이유가 설명이 된다.
        - 하지만 내가 다운받은 alpine linux에 busybox가 있는 것이고, 실제 우분투 환경에서는 busybox가 없다고 한다..경량화를 위해 container에서 쓰는 방식이기 때문에 우분투에 없다고 한다.
        - 어쨌든 궁금해서 busybox를 열어봤더니 엄청난 이진 데이터(숫자)가 있다!!
        - 그리고 명령어를 이용하여 ls, cat 등의 명령어를 볼 수 있었다.
        - `strings ./busybox | less`

## 본론

### Day 3 - cgroup을 사용해서 container의 자원 제한 및 할당하기

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

#### cgroup설정하기.

**- 내 cgroup version 확인해보기**

```text
sangyun8908@LAPTOP-OVR25738:/sys/fs/cgroup$ mount | grep cgroup
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate)
```

- cgroup version 2를 사용하고 있다는 것을 알 수 있다
- Todo: mount에 대해 공부하기

**- /sys/fs/cgroup에 내 container 설정 추가하기**

```text
sangyun8908@LAPTOP-OVR25738:/sys/fs/cgroup$ sudo mkdir my-container
```

- 일단 이 폴더 권한이 super user라는 것을 알 수 있고, 정말 신기하게 저렇게 디렉토리를 만들었더니 안에 필요한 파일들이 자동으로 생성되었다!!

```text
sangyun8908@LAPTOP-OVR25738:/sys/fs/cgroup/my-container$ ls
cgroup.controllers      cpu.stat.local         memory.max
cgroup.events           cpu.weight             memory.min
cgroup.freeze           cpu.weight.nice        memory.numa_stat
cgroup.kill             cpuset.cpus            memory.oom.group
cgroup.max.depth        cpuset.cpus.effective  memory.peak
cgroup.max.descendants  cpuset.cpus.partition  memory.pressure
cgroup.pressure         cpuset.mems            memory.reclaim
cgroup.procs            cpuset.mems.effective  memory.stat
cgroup.stat             io.latency             memory.swap.current
cgroup.subtree_control  io.max                 memory.swap.events
cgroup.threads          io.pressure            memory.swap.high
cgroup.type             io.stat                memory.swap.max
cpu.idle                memory.current         memory.swap.peak
cpu.max                 memory.events          pids.current
cpu.max.burst           memory.events.local    pids.events
cpu.pressure            memory.high            pids.max
cpu.stat                memory.low             pids.peak
```

- Todo: Cgroup에 대한 자세한 공부가 또 추가적으로 필요해 보인다...

**- 파일에 메모리 50MB로 제한 둬보기**

```text
sangyun8908@LAPTOP-OVR25738:/sys/fs/cgroup/my-container$ echo 50000000 | sudo tee /sys/fs/cgroup/my-container/memory.max
50000000
sangyun8908@LAPTOP-OVR25738:/sys/fs/cgroup/my-container$ echo $$ | sudo tee /sys/fs/cgroup/my-container/cgroup.procs
595
```

- 현재 쉘에 50mb제한을 걸었다, 한번 실험을 통해 제한이 되는지 확인해보자

```text
sangyun8908@LAPTOP-OVR25738:/sys/fs/cgroup/my-container$ dd if=/dev/zero of=/dev/null bs=1M count=100
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.00703325 s, 14.9 GB/s
```

- 왜 그러는지는 모르겠지만 아주 잘 된다...직접 프로그램을 작성해서 해보자

```go
package main

import "time"

func main() {
    // 100MB만큼 할당하고, 죽지 않게 대기시키기
    b := make([]byte, 100*1024*1024)
    for i := range b {b[i] = 1}
    time.Sleep(10*time.Minute)
}
```

- 계속 되지 않았지만, lunarvim이 확연히 느려지는 것을 확인하여 이유가 뭔지 원인을 좀 찾아보았다.
- 원인을 찾아보니 `memory.swap.max`라는 기능이 켜져있으면 컨테이너가 ram을 다 쓰고나서 하드디스크로 전환을 하는 일이 이루어져, 프로그램이 kill이 되지 않는다는 것을 알았다. 그래서 그 기능을 꺼보고 다시 해봤다

```bash
echo 0 | sudo tee /sys/fs/cgroup/my-container/memory.swap.max
```

- 이걸 실행하고 다시 main.go를 돌려보니 그냥 터미널이 꺼져버렸다!! 그리고 그 터미널의 이벤트를 추적하는 다른 터미널에서 kill 로그가 뜨는 것을 확인하였다.
- Todo: `tee` 명령어와 `dd` 명령어에 대한 공부가 필요해보인다.
- Todo: memory.swap기능에 대해 더 알아봐야겠다.

**- 진짜 내 컨테이너에게 자원 제한 두기**

- 이제는 진짜 내 컨테이너에게 자원 제한을 하는 일만 남았다.

```go
//cgroup.go
package cgroups

import (
    "os"
    "strconv"
    "fmt"
)
func SetCgroup(containerID string, pid int) error {
    cgroupPath := "/sys/fs/cgroup/container-" + containerID

    // sudo 로 실행되기 때문에 차피 다 권한이 있음

    // 폴더 생성(cgroup가상 파일시스템 폴더 생성, 이거 생성되면 안에서 알아서
    // 다른 것들도 만들어짐)
    if err := os.Mkdir(cgroupPath, 0755); err != nil && !os.IsExist(err) {
        return err
    }

    fmt.Println("로그: cgroup파일 만들어짐, 경로: %s", cgroupPath)
    // 메모리 제한 설정하기, 질리게 했던거임
    os.WriteFile(cgroupPath+"/memory.max", []byte("20000000"), 0644)

    // 스왑 제한하기
    os.WriteFile(cgroupPath + "/memory.swap.max", []byte("0"), 0644)

    // 프로세스 등록하기
    err := os.WriteFile(cgroupPath+"/cgroup.procs", []byte(strconv.Itoa(pid)), 0644)
    return err
}

// 파일 삭제를 위한 별도 함수
func RemoveCgroup(containerID string) error {
    cgroupPath := "/sys/fs/cgroup/container-" + containerID
    return os.Remove(cgroupPath)
}
```

- 일단 cgroup격리 파일을 만들고, 그 격리 파일 안의 값들을 `os.WriteFile`로 설정하도록 코드를 짰다. 또한, 컨테이너가 끝나면 만들었던 cgroup파일을 자동으로 없애도록 하기 위해 함수를 하나 더 만들었다

```go
//main.go
...

    // 컨테이너 실행
    if err := c.Start(); err != nil {
        fmt.Printf("컨테이너 실행 실패: %v\n", err)
        return
    }

    // cgroup 설정하기
    cgroups.SetCgroup("1", c.Cmd.Process.Pid)

    // 컨테이너 끝날때까지 기다리고, 끝나면 삭제하기
    defer cgroups.RemoveCgroup("1")

    // 프로세스 대기
    // 컨테이너 프로세스각 끝나야 끝남
    err := c.Cmd.Wait()
...
```

- 그리고 확실히 컨테이너가 끝나면 만들었던 cgroup파일을 지우는 `cgroups.RemoveCgroup()` 함수를 `defer`구문으로 실행되도록 보장했다

### Observation(실행 결과)

- 최종 실행 결과에 대해 이야기 해보겠다!

```text
sangyun8908@LAPTOP-OVR25738:~/go_programming/micro_container$ sudo ./micro_container /bin/sh
새로운 컨테이너 생성중,,,/bin/sh
컨테이너 ID: 1, PID: 4229 로 실행됨
로그: cgroup파일 만들어짐, 경로: %s /sys/fs/cgroup/container-1
/ #
```

- 일단 아까 `SetCgroup`에 넣어놓은 로그가 잘 뜨는 것을 볼 수 있다.

```text
sangyun8908@LAPTOP-OVR25738:/sys/fs/cgroup$ ls
cgroup.controllers      cpu.stat               memory.pressure
cgroup.max.depth        cpu.stat.local         memory.reclaim
cgroup.max.descendants  cpuset.cpus.effective  memory.stat
cgroup.pressure         cpuset.mems.effective  sys-fs-fuse-connections.mount
cgroup.procs            dev-hugepages.mount    sys-kernel-config.mount
cgroup.stat             dev-mqueue.mount       sys-kernel-debug.mount
cgroup.subtree_control  init.scope             sys-kernel-tracing.mount
cgroup.threads          io.pressure            system.slice
**container-1**             io.stat                user.slice
cpu.pressure            memory.numa_stat
```

- 또한, cgroup경로에 우리가 만든 container-1이 만들어진 것을 볼 수 있다!!
- exit로 container를 종료하면 container-1도 사라지는 것을 확인했다.

```text
sangyun8908@LAPTOP-OVR25738:~/go_programming/micro_container$ sudo ./micro_container /bin/sh
새로운 컨테이너 생성중,,,/bin/sh
컨테이너 ID: 1, PID: 4252 로 실행됨
로그: cgroup파일 만들어짐, 경로: %s /sys/fs/cgroup/container-1
/ # ls
bin    etc    lib    mnt    proc   run    srv    tmp    var
dev    home   media  opt    root   sbin   sys    usr
/ # dd if=/dev/zero of=/dev/shm/test bs=1M count=30
dd: can't open '/dev/zero': No such file or directory
/ # a="init"
/ # while true; do a="$a$a"; echo "현재 문자열 크기 증가 중..."; done
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
현재 문자열 크기 증가 중...
컨테이너가 에러남: signal: killed
```

- 마지막으로 메모리 부하 실험이다
- 문자열 크기를 계속 증가하다가 제한된 용량을 넘어서자 killed 로그가 뜨면서 멈춘 것을 볼 수 있다!

### Why?(실행 결과 분석)

### Trouble Shooting

- 많은 일이 있었다...`memory.swap` 설정 때문에 메모리 제한이 잘 안되었으며, 파일 구조를 뜯어고치는 바람에 파일 경로도 다 바꾸고, build명령어 치기 귀찮아서 Makefile도 만들고 별별 일이 있었다. 다시 한번 느끼지만 초반 구조를 잘 잡는게 정말 중요한 거 같다.

## 결론

- 이번 실습을 통해 단순히 컨테이너를 만드는 것이 아닌, 리눅스 커널이 어떻게 프로세스를 관리하는지를 경험할 수 있었다.
- 처음에는 단순하게 `memory.max`파일에 숫자를 적으면 자원이 제한될 줄 알았지만, 실제로는 커널의 **메모리 관리 정책(Swap)** 과 **프로세스 생명주기(defer)** 를 정확히 이해하고 있어야 시스템을 통제할 수 있다는 점을 알게 되었다.

### Key Insights(핵심 정리)

- Cgroups v2 제어: `/sys/fs/cgroup` 가상 파일 시스템을 통해 프로세스의 자원을 물리적으로 제한할 수 있다.
- 메모리와 스왑의 관계: `memory.max`로 메모리 제한하는 것만으로는 완벽한 제어가 되지 않는다. `memory.swap.max`를 0으로 설정해야 컨테이너가 호스트의 디스크로 도망가지 못하고 엄격하게 제한(OOM kill)된다.
- 생명주기 관리: `Cgroup`폴더는 프로세스가 종료된 후에도 남기 때문에, Go의 `defer`나 별도의 정리 로직을 통해 컨테이너 종료 시점에 반드시 삭제해줘야 시스템 오염을 막을 수 있다!
- 커널의 메커니즘: `memory.events`를 통해 커널이 프로세스에게 얼마나 많은 할당 제한을 걸었는지, 실제로 죽었는지를 추적할 수 있다.

### Next Step(다음 주제)

- 다음 일차에는 Network부분쪽에 격리를 해볼 예정이다

1. veth pair(가상 케이블): 컨테이너와 호스트를 잇는 가상 랜선을 만든다.
2. Bridge(공유기): 여러 컨테이너가 서로 통신할 수 있게 가상 스위치를 설계한다.
3. IP할당: 컨테이너에게 고유한 주소를 부여한다.

### TODO

- Cgroup에 대해 추가 공부하기(사용법 위주 말고 개념 위주로)
    - memory.~ 같은 설정들에 대해서도
- dd, tee 같은 unix명령어 다시 확인해보기

### github repo 주소

[https://github.com/Joseng8908/go_programming](https://github.com/Joseng8908/go_programming)
