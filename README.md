# FastHorse（快马）第二代

快马是一个 **TCP + 混淆 + 动态门 + TLS 1.3 + H2 CONNECT** 的代理协议，定位是 UDP 被封时的 TCP 兜底。

```
TCP
└── obfs（ChaCha20，时间窗口轮换）
    └── 动态门（时间戳 + nonce + HMAC）
        └── TLS 1.3
            └── H2 CONNECT（原生多流）
```

## 第二代的变化

第一代使用连接池：多条 H2 连接、按在途流选连接、定时补连接。第二代改为 **单条 H2 连接 + 原生多流**，把复用交回 H2，不再自己维护连接池。

| 能力 | 第一代 | 第二代 |
|---|---|---|
| 连接模型 | 多连接池 | 单 H2 连接，原生多流 |
| 并发控制 | 按在途流选连接 | 客户端流信号量 + 服务端 H2 并发流限制 |
| 连接切换 | 失败/降速触发换连接 | 只保留端口轮换，替换底层连接 |
| 限速 | `target_mbps` 目标值 | `up_mbps` / `down_mbps` 令牌桶，真实限速 |
| 端口轮换 | 定时 + 失败 + 降速 | 仅显式开启；默认关闭，端口数 <= 1 不轮换 |
| 客户端实现 | cronet / 连接池 | 纯 Go H2，无需 libcronet |
| 内核适配 | 无 | `type: "fasthorse"`，面板可下发 |

## 结构

```
cmd/            CLI
internal/       服务端、配置
pkg/client/     纯 Go 客户端（H2 多流、raw TLS、限速）
pkg/wire/       H2 CONNECT、流封装、TLS
pkg/obfs/       混淆
pkg/gate/       动态门
pkg/pool/       旧连接池（保留兼容）
```

## 构建

```bash
go build ./...
go build ./cmd/kuaima
```

不再需要 `libcronet`。

## 服务端

```json
{
  "listen": "0.0.0.0",
  "listen_port_range": "20000:20002",
  "certificate_path": "/etc/kuaima/cert.pem",
  "key_path": "/etc/kuaima/key.pem",
  "obfs": { "password": "change-me" },
  "gate": { "node_id": "hk-01" },
  "users": [
    { "username": "alice", "password": "change-me-too" }
  ]
}
```

```bash
kuaima serve -c server.json
```

## 客户端

```json
{
  "type": "fasthorse",
  "server": "1.2.3.4",
  "server_port": 20000,
  "server_ports": ["20000", "20001", "20002"],
  "username": "alice",
  "password": "change-me-too",
  "obfs": { "password": "change-me" },
  "gate": { "node_id": "hk-01" },
  "tls": { "enabled": true, "server_name": "kuaima", "insecure": true },
  "max_streams_per_connection": 64,
  "port_hop_enabled": false,
  "port_rotate_seconds": 120,
  "up_mbps": 50,
  "down_mbps": 300
}
```

- `max_streams_per_connection`：客户端并发流信号量，服务端 H2 `MaxConcurrentStreams` 同步限制；
- `up_mbps`：客户端聚合上行限速；
- `down_mbps`：服务端发往该客户端连接的聚合下行限速；
- `port_hop_enabled` 不写或 `false` 都不轮换；端口数 <= 1 强制不轮换。

## 吞吐策略

快马跑在系统 TCP 上，拥塞控制由操作系统决定，协议层不提供 Brutal：

- 客户端 H2 单连接，所有 CONNECT 流共享同一条 TCP；
- 上行在客户端按 `up_mbps` 令牌桶限速；
- 下行在服务端按 `down_mbps` 对整条 H2 连接限速；
- H2 服务端接收窗口：连接 / 单流 3 MB，帧 1 MB；
- 客户端 TLS 收发缓冲 4 MB；
- 端口轮换只替换底层连接，默认关闭。

## 已完成

- 纯 Go H2 CONNECT，客户端无需 cronet；
- 专用 TLS + `FHMX` 魔数，避免 H2 CONNECT 承载 mux 的流取消问题；
- 服务端：端口段监听、Basic 用户校验、目标策略、每 IP 连接数；
- 客户端：单连接多流、流并发上限、失败重连；
- 限速：客户端上行、服务端下行；
- 端口轮换：显式开启，默认关闭；
- sing-box 适配：`type: "fasthorse"`，面板/订阅可下发。

## 限制

- TCP 拥塞控制由系统决定，无法按连接指定 BBR/Brutal；
- 单连接共享 TCP 拥塞窗口，丢包时所有流一起降速；
- 自签证书有效期不超过 398 天；
- 内层 Basic 是静态密码，动态性由外层门提供；
- 无配额和每用户限速；
- 没有 `connections` / `target_mbps` 字段：已由单连接多流和上下行限速取代。

## 许可

见 [LICENSE](LICENSE)。
