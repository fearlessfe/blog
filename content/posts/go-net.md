---
title: "Go Net"
date: 2022-07-02T22:40:13+08:00
draft: true
---

```go
listen, err := net.Listen("tcp", endpoint)
```

```go
// net.dial.go
func Listen(network, address string) (Listener, error) {
	var lc ListenConfig
	return lc.Listen(context.Background(), network, address)
}
```