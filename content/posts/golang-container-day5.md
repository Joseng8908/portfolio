---
title: "(5일차) Golang으로 나만의 컨테이너 만들어보기"
date: 2026-03-29T17:08:19+09:00
lastmod: 2026-03-29T17:08:19+09:00
draft: false
tags: ["Go", "container", "linux", "docker", "networking"]
summary: "netlink 라이브러리로 브릿지 생성과 veth 연결을 코드로 자동화한다."
ShowToc: true
TocOpen: false
---

## 서론

- 저번 일차 복습
    - CLI상에서 Linux명령어를 통해 veth같은 가상 랜선을 만들고, 호스트와 컨테이너를 연결하고, UP시키는 작업을 해 보았다.
- TODO
    - 없었음.

## 본론

### Day 5 - 네트워크 연결 및 브릿지 생성 자동화 구현하기

- 오늘 할 내용이 매우 복잡하기 때문에 미리 정리를 해두고 포스팅을 시작하도록 하겠다.
    1. veth(virtual ethernet) 가상 랜선 만들기.
    2. 브릿지(네트워크에 있는 일종의 스위치, 라우터) 만들기.
    3. 컨테이너 만들기.
    4. 브릿지와 컨테이너에 veth를 연결하기.
    5. 연결한 veth을 브릿지와, 컨테이너 기준으로 각각 활성화 시키기.

### 기본적인 파일 구조

```text
micro-docker/
    - cmd/
        - conatiner/
            - main.go # 프로그램 진입점
    - internal/
        - cgroups/
            - cgroups.go # 자원 격리 로직
        - container/
            - config.go # 컨테이너 관련 객체 선언
            - child.go # 컨테이너 내부 설정 및 실행
            - run.go # 컨테이너 실행 전 세팅
        - network/
            - bridge.go # bridge만들고, 가상랜선 연결 및 활성화시키는 로직
            - config.go # 컨테이너 쪽 가상 랜선 연결하고, 활성화시키는 로직
            - veth.go # 가상 랜선 만들기
```

- 전체적인 리팩토링을 거쳐 파일 구조가 조금 달라졌다!

### Code

#### 네트워크 격리하기 ver2.

**- bridge 생성하기**

```go
// bridge.go
package network

import (
    "fmt"
    "net"

    "github.com/vishvananda/netlink"
)

func GetOrCreateBridge(bridgeName string) error {

    // 지정하려는 브릿지의 이름이 이미 있는지 확인하는 로직
    // 호스트의 모든 네트워크 장치 가져오기
    // 이름으로 장치 찾아보기
    link, err := netlink.LinkByName(bridgeName)

    // 만약 err가 안뜨면 장치가 존재한다는 뜻이니까 그 장치가 bridge인지 확인하고
    // 그대로 재사용 하기
    if err == nil {
        if link.Type() == "bridge" {
            fmt.Println("log: 이미 존재하는 브릿지입니다. 그대로 재사용하겠습니다.")
            // 브릿지 활성화되어 있는지 확인하기
            if IsBridgeUp(bridgeName) {
                return nil
            }
            fmt.Println("log: 브릿지가 꺼져 있어 활성화 합니다")
            return netlink.LinkSetUp(link)
        }
        return fmt.Errorf("log: 이름은 같지만 브릿지 타입이 아닙니다.") 
    }

    // 만약 err가 뜨면 장치가 없다는 뜻이니까 정상
    // 새로 브릿지 만들기
    // 브릿지 상세 설정 객체 생성하기
    // 객체 이름: linkattributes
    la := netlink.NewLinkAttrs()
    la.Name = bridgeName

    // 브릿지 전용 객체 만들기
    br := &netlink.Bridge{LinkAttrs: la}

    // 커널에 브릿지 추가 요청(즉 스위치 만들어달라고 요청하는거임)
    if err := netlink.LinkAdd(br); err != nil{
        return err
    }

    // 브릿지에 ip주소 부여하기, 즉 스위치 주소 지정해주는 것임
    addr, _ := netlink.ParseAddr("172.17.0.1/24") //서브넷 주소 1로

    // 생성한 브릿지에 ip주소 할당하기ㅣ
    if err := netlink.AddrAdd(br, addr); err != nil {
        return err
    }

    return netlink.LinkSetUp(br) // 브릿지 활성화 시키기! 
}

// 브릿지가 활성화되어 있는지 확인하는 메소드
func IsBridgeUp(bridgeName string) bool{
    link, err := netlink.LinkByName(bridgeName)
    if err != nil {
        return false
    }

    isUp := link.Attrs().Flags & net.FlagUp != 0

    return isUp
}
```

- 일단 브릿지를 생성하기 전에, 기존의 같은 이름의 브릿지가 있으면 활성화하고, 없으면 생성하도록 코드를 짰다.
- 이 부분은 크게 어렵지 않았다. 단지 브릿지를 만들고, ip주소를 할당하고, 활성화 시키는 것이 전부이다.
- TODO: 왜 여기서 그냥 br만 만들지 않고, LinkAttrs객체를 만들고, 그를 활용해서 bridge객체를 만드는걸까?

**- veth(가상 랜선) 만들기**

```go
// veth.go
package network

import (
    "fmt"

    "github.com/vishvananda/netlink"
)

// pid는 container의 pid임
func SetupVeth(pid int, bridgeName string) error {
    // 호스트쪽 랜선? 이름 정의
    hostVethName := fmt.Sprintf("veth%d", pid)
    // 컨테이너 쪽에서는 eth0-temp으로 보이게 정의
    containerVethName := "eth0-temp" 

    // veth 쌍 설정하기,,,,
    veth := &netlink.Veth{
        LinkAttrs: netlink.LinkAttrs{Name: hostVethName},
        PeerName: containerVethName,
    }

    // 커널에 랜선 만들자고 요청하기
    if err := netlink.LinkAdd(veth); err != nil {
        return fmt.Errorf("veth 생성 실패: %v", err)
    }

    // 브릿지에 끼울 랜선 가져오기
    hostVeth, _ := netlink.LinkByName(hostVethName)
    // 브릿지 장치 가져오기
    br, _ := netlink.LinkByName(bridgeName)

    // 호스트쪽 랜선 활성화 (UP시키기)
    if err := netlink.LinkSetMaster(hostVeth, br); err != nil {
        return fmt.Errorf("브릿지 쪽 veth 활성화 실패: %v", err)
    }

    // 호스트 쪽 남아있는 vetHost UP시키기
    if err := netlink.LinkSetUp(hostVeth); err != nil {
        return fmt.Errorf("호스트 veth 활성화 실패: %v", err)
    }

    // 컨테이너에 끼울 랜선 가져오기
    peer, err := netlink.LinkByName(containerVethName)
    if err != nil {
        return fmt.Errorf("컨테이너 veth 찾기 실패: %v", err)
    }

    // 컨테이너의 네임스페이스로 이동
    if err := netlink.LinkSetNsPid(peer, pid); err != nil {
        return fmt.Errorf("네임스페이스로 이동 실패: %v", err)
    }

    fmt.Printf("log: %s 장치를 PID %d로 이동 완료\n", containerVethName, pid)

    return nil

}
```

- 이 부분이 참 어려웠다..
- 일단 veth을 보는 쪽의 관점에 따라 이름을 각각 정의해줘야 한다.
- 호스트(bridge) 쪽에서는 랜선을 veth+컨테이너의 pid 로 인식하게 했다
- 컨테이너 쪽에서는 랜선을 eth0-temp로 인식하게 했다.

```go
// 호스트쪽 랜선? 이름 정의
    hostVethName := fmt.Sprintf("veth%d", pid)
    // 컨테이너 쪽에서는 eth0-temp으로 보이게 정의
    containerVethName := "eth0-temp" 
```

- 그런 후, veth의 양쪽을 정의해 준다

```go
    veth := &netlink.Veth{
        LinkAttrs: netlink.LinkAttrs{Name: hostVethName},
        PeerName: containerVethName,
    }
```

- 그런 후 커널에 랜선을 만들라고 요청하고, **브릿지** 기준에서 먼저 랜선을 끼운다.
- 그리고 UP을 시켜야하는데, 처음에 LinkSetMaster만 하면 되는 줄 알았는데, 이걸 하고, LinkSetUp까지 해야한다는 사실을 알았다..
- TODO: LinkSetMaster와 LinkSetUp 알아보기

```go
// 브릿지에 끼울 랜선 가져오기
    hostVeth, _ := netlink.LinkByName(hostVethName)
    // 브릿지 장치 가져오기
    br, _ := netlink.LinkByName(bridgeName)

    // 호스트쪽 랜선 활성화 (UP시키기)
    if err := netlink.LinkSetMaster(hostVeth, br); err != nil {
        return fmt.Errorf("브릿지 쪽 veth 활성화 실패: %v", err)
    }

    // 호스트 쪽 남아있는 vetHost UP시키기
    if err := netlink.LinkSetUp(hostVeth); err != nil {
        return fmt.Errorf("호스트 veth 활성화 실패: %v", err)
    }
```

- 그리고 이제 컨테이너 쪽에서 설정을 해줘야 한다.
- 여기서 랜선을 가져오고, 이 랜선을 컨테이너로 **던져**줘야 한다!
- 그리고 컨테이너로 들어가서, 이 던진 랜선을 받고, 활성화시켜줘야 한다...

```go
// 컨테이너에 끼울 랜선 가져오기
    peer, err := netlink.LinkByName(containerVethName)
    if err != nil {
        return fmt.Errorf("컨테이너 veth 찾기 실패: %v", err)
    }

    // 컨테이너의 네임스페이스로 이동
    if err := netlink.LinkSetNsPid(peer, pid); err != nil {
        return fmt.Errorf("네임스페이스로 이동 실패: %v", err)
    }
```

- 위의 로직이 랜선을 컨테이너에게 던지고, 컨테이너에 들어가는 것까지 한 것이다.
- 이제 컨테이너 내에서의 설정 코드를 보도록 하자.

**- 컨테이너 안에서 랜선 설정하기**

```go
// config.go
package network

import (
    "fmt"
    "time"
    "github.com/vishvananda/netlink"
)

// 여기서 pid는 container의 pid
func ConfigureContainerNetwork(ifaceName string) error {
    // 이미 네임스페이스 안에서 실행하므로 호스트에서 들어갈 열쇠 구하고,,,등등의 과정 필요 없음
    // 컨테이너 안에서 세팅하기
    var eth0 netlink.Link
    var err error

    // 최대 5번 재시도 (부모가 선을 던져줄 때까지 대기)
    for i := 0; i < 5; i++ {
        eth0, err = netlink.LinkByName(ifaceName)
        if err == nil {
            break
        }
        fmt.Printf("장치 %s 대기 중... (%d/5)\n", ifaceName, i+1)
        time.Sleep(100 * time.Millisecond)
    }

    if err != nil {
        return fmt.Errorf("결국 장치를 찾지 못했습니다: %v", err)
    }

    // 컨테이너에게 ip주소 부여하기(1712.17.0.2)
    addr, _ := netlink.ParseAddr("172.17.0.2/24")
    if err := netlink.AddrAdd(eth0, addr); err != nil {
        return err
    }

    // eth0 UP시키기
    if err := netlink.LinkSetUp(eth0); err != nil {
        return err
    }

    lo, _ := netlink.LinkByName("lo")
    netlink.LinkSetUp(lo)
    return nil
}
```

- 이 부분에서 처음에는 컨테이너 안에서 시작하는 것이 아닌, 브릿지에서 시작을 하여 컨테이너 내부로 들어가야 해서 키를 받고, 그 키를 활용해 컨테이너를 들어가는 로직이 있었다. 하지만 현재 veth.go에서 우리는 컨테이너 내부에 이미 들어왔기 때문에 그럴 필요가 없어졌다.
- 이 부분에서 참 애를 많이 먹었다. 자세한 내용은 Trouble Shooting쪽에서 다루도록 하겠다.
- TODO: lo부분 이해하기

### Observation(실행 결과)

- 암튼 위의 세 로직을 짜고, 이러쿵 저러쿵 잘 조합한 결과 컨테이너 내/외부와의 네트워크 연결에 성공했다!

```text
╰─ sudo ./micro-container run /bin/bash                                                            ─╯
부모 프로세스 시작(PID: 0)
로그: cgroup파일 만들어짐, 경로: %s /sys/fs/cgroup/container-test-container
자식 프로세스(컨테이너) 시작 (PID: 1)log: eth0-temp 장치를 PID 1439로 이동 완료
root@micro-container:/home/sangyun8908/go_programming/micro-container# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
7: eth0-temp@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 26:10:c5:a7:1c:e8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/24 brd 172.17.0.255 scope global eth0-temp
       valid_lft forever preferred_lft forever
    inet6 fe80::2410:c5ff:fea7:1ce8/64 scope link
       valid_lft forever preferred_lft forever
root@micro-container:/home/sangyun8908/go_programming/micro-container#
```

- eth0-temp부분의 결과를 보면 UP 상태라는 것을 볼 수 있다!!!
- 한번 ping으로 브릿지와의 연결도 잘 되었는지 확인해보자.

```text
root@micro-container:/home/sangyun8908/go_programming/micro-container# ping -c 3 172.17.0.1
PING 172.17.0.1 (172.17.0.1) 56(84) bytes of data.
64 bytes from 172.17.0.1: icmp_seq=1 ttl=64 time=0.243 ms
64 bytes from 172.17.0.1: icmp_seq=2 ttl=64 time=0.056 ms
64 bytes from 172.17.0.1: icmp_seq=3 ttl=64 time=0.053 ms

--- 172.17.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2037ms
rtt min/avg/max/mdev = 0.053/0.117/0.243/0.088 ms
```

- 보면 우리가 지정한 브릿지의 주소인 172.17.0.1에서 응답이 3 received로 뜬 것을 볼 수 있다.

### Why?(실행 결과 분석)

- 실행 결과를 한번 분석해보자.
- 일단 `sudo ./micro-container run /bin/sh`로 컨테이너를 실행한다.
- 이때 실행되는 것을 순차적으로 보자면
    1. 컨테이너 cgroup 및, 내부 네트워크, 네임스페이스, 루트 파일시스템 초기화.
    2. 브릿지 생성 및, veth만들고, 브릿지 부분에서 활성화 시키고 container쪽에게 랜선 던지기.
    3. 컨테이너 쪽에서 랜선 받고, 활성화 시키기.
- 이정도로 볼 수 있다.
- 여기서 `ip addr`를 컨테이너 쪽에서 치면 현재 연결되어 있는 랜선에 대한 내용들이 나온다.
    - lo는 자기 자신을 뜻하고, eth0-temp가 우리가 만든 것이다!
    - 로그를 더 자세히 보자면 UP상태로 잘 돌아가고 있다는 것을 알 수 있다.
    - TODO: 로그 분석 더 자세히 해보기.
- 또한 ping을 쳐서 호스트(bridge)가 잘 살아있는지 확인해본다.
    - 3 packets transmitted, 3 received 로그가 뜬 것을 보면 잘 왔다는 사실을 알 수 있다.
    - TODO: 여기 부분도 로그 분석 더 자세히 해보기.

### Trouble Shooting

- config.go 쪽에서 많은 어려움이 있었다.
- 일단 우리의 레이어를 생각해보면 wsl에서 bridge프로세스와 container프로세스를 만들고, bridge와 container를 연결해야 하는데 이 부분의 순서가 꼬여서 Sleep을 많이 활용했다.
- TODO: 어라 이 부분,,,고루틴으로 해결이 안되나??? Context Switching을 잘 하면 될 거 같기도 하는데....

## 결론

### Key Insights(핵심 정리)

- 우리가 Go로 짠 코드는 단지 리눅스 커널의 Namespace, Cgroups, Netlink 기능을 호출할 뿐이었다. 결국 컨테이너 기술은 기존 OS 커널 기능을 어떻게 잘 활용할까에 대한 싸움이다.
- veth pair는 마치 가상 랜선의 양 끝단처럼 사용해서, 한쪽을 브릿지에 꽂고 한쪽을 컨테이너에 던져주는 것만으로 물리적인 네트워크 연결을 흉내낼 수 있다.

### Next Step(다음 주제)

- OverlayFs

### TODO

- LinkAttrs를 따로 만드는 이유
- LinkSetMaster vs LinkSetUp
- lo란?
- context switching을 이용해서 브릿지, 가상랜선, 컨테이너 생성 비동기처리

### github repo 주소

[https://github.com/Joseng8908/go_programming](https://github.com/Joseng8908/go_programming)
