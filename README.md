# ![OpenGFW](docs/logo.png)

[![License][1]][2]

[1]: https://img.shields.io/badge/License-MPL_2.0-brightgreen.svg

[2]: LICENSE

**[中文文档](README.zh.md)**

OpenGFW is a flexible, easy-to-use, open source implementation of [GFW](https://en.wikipedia.org/wiki/Great_Firewall) on
Linux that's in many ways more powerful than the real thing. It's cyber sovereignty you can have on a home router.

> [!CAUTION]
> This project is still in very early stages of development. Use at your own risk.

> [!NOTE]
> We are looking for contributors to help us with this project, especially implementing analyzers for more protocols!!!

## Features

- Full IP/TCP reassembly, various protocol analyzers
    - HTTP, TLS, DNS, SSH, and many more to come
    - "Fully encrypted traffic" detection for Shadowsocks,
      etc. (https://gfw.report/publications/usenixsecurity23/data/paper/paper.pdf)
    - [WIP] Machine learning based traffic classification
- Flow-based multicore load balancing
- Connection offloading
- Powerful rule engine based on [expr](https://github.com/expr-lang/expr)
- Flexible analyzer & modifier framework
- Extensible IO implementation (only NFQueue for now)
- [WIP] Web UI

## Use cases

- Ad blocking
- Parental control
- Malware protection
- Abuse prevention for VPN/proxy services
- Traffic analysis (log only mode)

## Usage

### Build

```shell
go build
```

### Run

```shell
export OPENGFW_LOG_LEVEL=debug
./OpenGFW -c config.yaml rules.yaml
```

### Example config

```yaml
io:
  queueSize: 1024
  local: true # set to false if you want to run OpenGFW on FORWARD chain

workers:
  count: 4
  queueSize: 16
  tcpMaxBufferedPagesTotal: 4096
  tcpMaxBufferedPagesPerConn: 64
  udpMaxStreams: 4096
```

### Example rules

Documentation on all supported protocols and what field each one has is not yet ready. For now, you have to check the
code under "analyzer" directory directly.

For syntax of the expression language, please refer
to [Expr Language Definition](https://expr-lang.org/docs/language-definition).

```yaml
- name: block v2ex http
  action: block
  expr: string(http?.req?.headers?.host) endsWith "v2ex.com"

- name: block v2ex https
  action: block
  expr: string(tls?.req?.sni) endsWith "v2ex.com"

- name: block shadowsocks
  action: block
  expr: fet != nil && fet.yes

- name: v2ex dns poisoning
  action: modify
  modifier:
    name: dns
    args:
      a: "0.0.0.0"
      aaaa: "::"
  expr: dns != nil && dns.qr && any(dns.questions, {.name endsWith "v2ex.com"})
```

#### Supported actions

- `allow`: Allow the connection, no further processing.
- `block`: Block the connection, no further processing. Send a TCP RST if it's a TCP connection.
- `drop`: For UDP, drop the packet that triggered the rule, continue processing future packets in the same flow. For
  TCP, same as `block`.
- `modify`: For UDP, modify the packet that triggered the rule using the given modifier, continue processing future
  packets in the same flow. For TCP, same as `allow`.