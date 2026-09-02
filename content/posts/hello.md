---
title: "Hello: What This Archive Is For"
date: 2026-09-02T09:00:00+09:00
lastmod: 2026-09-02T09:00:00+09:00
draft: false
tags: ["meta", "linux", "scheduling"]
summary: "A placeholder first post — also a smoke test for syntax highlighting, the table of contents, and tag routing."
ShowToc: true
TocOpen: false
---

This is a placeholder post. It exists so the build has something to render, and
so every feature the site claims to have is visible on one page: the table of
contents on the right, reading time in the header, syntax highlighting with line
numbers, a working copy button, and three tags routing into the tag index.

Replace it when the first real post lands.

## What gets written here

Notes with a measurement in them. Not tutorials, not link roundups — the
specific thing that was surprising, the command that showed it, and the
conclusion that survived.

Rough categories:

- **Kernel and scheduling** — CFS/EEVDF behavior, cgroup throttling, latency tails
- **Memory** — NUMA locality, page cache behavior, hugepages
- **Networking** — RDMA, kernel bypass, socket tuning, fabric topology
- **Orchestration** — Kubernetes runtime behavior, CNI paths, resource accounting

## Smoke test: code rendering

Go, because that is where most of my tooling lives:

```go
// Pin the current goroutine's OS thread to a specific NUMA node's CPU set.
// This is the cheap version — sched_setaffinity, no libnuma dependency.
func pinToNode(node int) error {
	cpus, err := cpusForNode(node)
	if err != nil {
		return fmt.Errorf("pinToNode(%d): %w", node, err)
	}

	runtime.LockOSThread()

	var set unix.CPUSet
	for _, cpu := range cpus {
		set.Set(cpu)
	}

	if err := unix.SchedSetaffinity(0, &set); err != nil {
		return fmt.Errorf("pinToNode(%d): sched_setaffinity: %w", node, err)
	}
	return nil
}
```

And a shell block, to confirm a second lexer works:

```bash
# Which node is this process actually allocating from?
numastat -p "$(pgrep -f my-service)"

# Per-node free memory, as the kernel sees it
numactl --hardware | grep -E 'node [0-9]+ (size|free)'
```

Inline code renders like `sched_setaffinity(2)` — no line numbers, no copy
button, which is correct.

### Nested heading

A third level, so the table of contents has something to nest. The ToC is
generated from `h2`–`h4` and collapsed by default; open it from the header of
this post.

## Next

Nothing yet. That is the point of a placeholder.
