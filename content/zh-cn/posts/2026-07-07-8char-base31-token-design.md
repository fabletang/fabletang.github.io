+++
title= "8位UUID, Base31 编码方案设计"
description= "短码设计"
date = "2026-07-07"
tags = [
    "golang",
    "uuid"
]
categories = [
  "技术"
]
series = [
  "概念"
]
+++

# 8位UUID, Base31 编码方案设计


**版本** 1.0
**日期** 2026-07-07

---

## 1. 设计目标

| 目标 | 说明 |
|---|---|
| **8 位固定长度** | 短小精悍，适合标签打印和二维码嵌入 |
| **Base31 编码** | 排除易混淆字符，提升人工输入准确率 |
| **二维码友好** | 短码可嵌入最小版本二维码（Version 1, 21×21） |
| **人工易输入** | 无大小写歧义，无数字字母混淆 |
| **概率唯一** | 基于种子哈希的概率唯一性 + 数据库 UNIQUE 约束兜底 |

### 适用场景

- **二维码令牌**：嵌入二维码，扫描后查找对应业务实体
- **短期操作凭证**：入库确认、验收签收等一次性操作
- **不适用场景**：资产编号（推荐使用 PRD 定义的 `ASSET+日期+流水`）、合同编号等需要排序和日期可读性的场景

---

## 2. 字符集设计

### 2.1 完整字符集

从标准字母数字集合（A-Z, 0-9 = 36 个字符）中剔除易混淆字符：

| 被排除 | 原因 | 易混淆为 |
|---|---|---|
| `0` | 数字 0 | `O` / `D` |
| `O` | 大写 O | `0` / `D` |
| `1` | 数字 1 | `I` / `L` |
| `I` | 大写 I | `1` / `L` |
| `L` | 大写 L | `1` / `I` |

保留字符（31 个）：

```
2  3  4  5  6  7  8  9
A  B  C  D  E  F  G  H
J  K  M  N  P  Q  R  S
T  U  V  W  X  Y  Z
```

### 2.2 为什么选择 31 个字符

| 基数 | 8 位容量（有效比特） | 剔除易混淆字符 | 评价 |
|---|---|---|---|
| Base36 | 41.4 bit | 否 | 包含 `0/O/1/I/L`，人工输入易错 |
| Base32 (RFC 4648) | 40.0 bit | 否（含 `O/0/1/L`） | 标准编码，但字符集对人工不友好 |
| Base32 (Crockford) | 40.0 bit | 部分（无 `I/L/O`，但有 `0/1`） | 接近理想，含数字 0/1 仍有混淆风险 |
| **Base31** | **39.6 bit** | **完全** | 所有易混淆字符全部剔除 |

31 是剔除 5 个易混淆字符后自然剩余的数量。容量损失极小（Base32 → Base31 仅损失 ~0.4 bit），换取了零歧义的人工输入体验。

### 2.3 容量计算

```
31^8 = 8,528,530,058,560 = 8.53 * 10^12 = 39.63 bit
```

实际使用的 39-bit 输入空间：

```
2^39 = 549,755,813,888 = 5.50 * 10^11
利用率 = 39.0 / 39.63 = 98.4%
```

39 bit 占用 8 位编码空间的 6.4%，留有充裕扩展余量（但无法超过 39.63 bit 天花板）。

---

## 3. 编码算法设计

### 3.1 算法流程

```
输入: (无参数)
输出: token string (8 字符)

Step 1 — 构造种子
    monotonic = runtime.nanotime() - processStartNano     // 单调递增，不受 NTP 影响
    random    = crypto.Read([8]byte)                        // CSPRNG 64 bit 真随机
    seed      = binary.LittleEndian.Uint64(random) ^ uint64(monotonic)

Step 2 — 哈希打散
    hash = CityHash64(seed)       // CityHash 提供优秀的雪崩效应

Step 3 — 截取
    bits = hash & 0x7F_FFFF_FFFF  // 取低 39 bit

Step 4 — Base31 编码
    token = Base31Encode(bits)    // 固定 8 字符，左填充 '2'
```

### 3.2 Seed 组成详解

| 组件 | 来源 | 熵贡献 | 说明 |
|---|---|---|---|
| **monotonic_nanos** | `runtime.nanotime()` - 进程启动时间 | ~0.03 bit/ns | 单调递增，不受 NTP 时钟跳变影响，防止时钟回拨导致种子重复 |
| **crypto/rand** | CSPRNG | 64 bit | 真随机性来源，保证多实例并行生成时无冲突 |
| ~~UTC 日期~~ | *已移除* | 0 bit | 对同一天的生成无熵贡献，被 monotonic_nanos 完全覆盖 |

相较于原方案，移除了 UTC 日期。理由：

- UTC 日期的精度为 1 天，对于同一天的生成调用贡献 0 额外熵
- `monotonic_nanos` 已经提供了更细粒度的时间信息
- 减少一次系统调用和字符串格式化开销

### 3.3 CityHash64 选择理由

| 属性 | CityHash64 | 替代方案 |
|---|---|---|
| 雪崩效应 | 优秀 — 1 bit 输入翻转导致 ~32 bit 输出翻转 | SipHash 同样优秀（但用于安全场景） |
| 吞吐量 | ~1.5 GB/s（单核） | XXH64 更快但质量略逊 |
| 实现复杂度 | ~300 行纯 Go，无 CGO | 可直接内嵌，零依赖 |
| 输出分布 | 均匀 — chi-squared 检验通过 | — |

**为什么不选 SipHash**：SipHash 设计目标是防 Hash DoS 攻击（带密钥），速度慢约 3 倍。本方案不需要加密强度哈希，CityHash 的速度优势更关键。

**为什么不选 XXH64**：XXH64 在单字节输入下雪崩效应较弱。本方案的种子仅 16 字节，CityHash 在短输入上表现更稳定。

### 3.4 编码表

```go
const base31Chars = "23456789ABCDEFGHJKMNPQRSTUVWXYZ"
```

编码方式：除基取余法（类 Base64 编码），结果反转后左填充至 8 位。

```go
func Base31Encode(n uint64) string:
    if n == 0: return "22222222"
    buf[8]byte
    i = 7
    while n > 0 and i >= 0:
        buf[i] = base31Chars[n % 31]
        n /= 31
        i--
    // 左填充 '2'
    for j = 0; j <= i; j++:
        buf[j] = '2'
    return string(buf[:])
```

---

## 4. 唯一性分析

### 4.1 容量与碰撞概率

39 bit 空间总计 549,755,813,888 个不同值。

基于生日悖论，在生成 N 个随机码后，至少出现一次碰撞的概率：

```
P(collision) ~ 1 - exp(-N^2 / (2 * 2^39))
```

| 生成数量 N | 碰撞概率 | 说明 |
|---|---|---|
| 1,000 | ~0.00009% | 可忽略 |
| 10,000 | ~0.009% | 可忽略 |
| 100,000 | ~0.9% | 约每 110 次插入需 1 次重试 |
| 500,000 | ~20% | 约每 5 次插入需 1 次重试 |
| **740,000** | **~50%** | 碰撞概率的分水岭 |
| 1,000,000 | ~60% | 每次插入约需 1.5 次重试 |
| 2,000,000 | ~97% | 重试开销开始不可接受 |
| 5,000,000 | ~99.9999% | 方案基本不可用 |

### 4.2 容量规划建议

| 场景规模 | 建议 |
|---|---|
| < 500,000（总码数） | 直接使用，无需额外措施 |
| 500,000 ~ 1,000,000 | 监控碰撞率，日志告警 |
| > 1,000,000 | 强烈不建议。改用 9 位 Base31（~45 bit）或 8 位 + 更高基数编码 |
| > 2,000,000 | 禁用此方案 |

### 4.3 为什么需要数据库 UNIQUE 约束

CityHash64 是确定性函数：相同的种子输入产生相同的输出。在以下场景中，两个不同请求可能产生相同的种子：

1. 同一纳秒内，多个 goroutine 读取 `crypto/rand` 的不同字节 → 产生不同种子（安全）
2. `crypto/rand` 调用失败（极度罕见），monotonic_nanos 相同 → 种子相同
3. 两个不同种子经过 CityHash64 后恰好低 39 bit 相同 → 概率性碰撞

数据库 `UNIQUE` 约束作为最后防线：

- INSERT 时使用 `ON CONFLICT (token) DO NOTHING RETURNING id`
- 受影响的行数为 0 时，重新生成（最多重试 10 次）
- 10 次重试后仍然冲突，返回明确错误

### 4.4 重试策略（两层保障）

```
// Layer 1: 内存去重（测试/benchmark 环境）
if inMemorySet.exists(token):
    retry++

// Layer 2: 数据库 UNIQUE 约束（生产环境）
INSERT INTO tokens (code) VALUES ($1) ON CONFLICT (code) DO NOTHING RETURNING id
if no rows returned:
    retry++
```

---

## 5. 性能分析

### 5.1 单次生成耗时

| 步骤 | 耗时 | 占比 |
|---|---|---|
| `runtime.nanotime()` | ~0.02 us | < 0.1% |
| `crypto/rand` (8 bytes) | 1–5 us | 60–80% |
| CityHash64 | ~0.05 us | ~1% |
| Base31 编码 | ~0.1 us | ~2% |
| **总计（生成）** | **~2–6 us** | — |
| 数据库 INSERT + UNIQUE 检查 | ~1 ms | 瓶颈 |
| **总计（含 DB）** | **~1 ms** | — |

### 5.2 吞吐量估算

| 场景 | 吞吐量 | 瓶颈 |
|---|---|---|
| 纯生成（无冲突检查） | ~170K–500K/s 单核 | `crypto/rand` 速率 |
| 生成 + 内存去重（N=1M） | ~200K/s 单核 | ~1.5 次平均重试 |
| 生成 + DB 写入 | ~1,000/s 单连接 | 数据库 I/O |
| 生成 + DB + 连接池（10 连接） | ~8,000/s | 数据库并发能力 |

### 5.3 多实例部署

多实例并行生成时碰撞概率增加，但分布式特性不会引入新风险：

- 各实例独立调用 `crypto/rand`，种子天然不相关
- 碰撞概率仅取决于**总生成数量**，与实例数目无关
- 数据库 UNIQUE 约束跨实例统一生效

| 实例数 | 每实例码数 | 总码数 | 碰撞概率 |
|---|---|---|---|
| 1 | 100,000 | 100,000 | ~0.9% |
| 2 | 100,000 | 200,000 | ~3.5% |
| 5 | 100,000 | 500,000 | ~20% |
| 10 | 50,000 | 500,000 | ~20% |
| 20 | 50,000 | 1,000,000 | ~60% |

### 5.4 优化建议

若 `crypto/rand` 成为瓶颈（>1M/s 需求），可使用缓冲区池：

```go
type RandPool struct {
    pool chan []byte
}

func (r *RandPool) Read(p []byte) {
    buf := <-r.pool
    copy(p, buf)
    r.pool <- buf
}
```

预读取放大因子 8x，将 `crypto/rand` 调用次数压缩至 1/8。

---

## 6. 安全分析

### 6.1 可预测性

| 攻击向量 | 难度 | 说明 |
|---|---|---|
| 从已观察码值反推种子 | 不可行 | CityHash 是单射函数，无法逆推；即使反推出种子，`crypto/rand` 每个种子都不同 |
| 暴力枚举 2^39 空间 | ~17 年/1000 QPS | 实际不可行，但空间只有 39 bit，不抗超级计算机 |
| 生日攻击（寻找碰撞） | O(2^(39/2)) = O(2^19.5) = 740K 次尝试 | 若有观察到的已知码，理论可找到碰撞。但攻击者无利可图 — 没有会话劫持、权限提升等价值 |
| 时序攻击（猜测下一码） | 不可行 | 每码独立随机生成，无序列关系 |

**结论**：39 bit 空间不满足加密安全（cryptographic security）要求（通常需要 >=128 bit），但对于 WMS 二维码令牌场景完全足够。攻击者没有经济利益动机进行碰撞攻击。

### 6.2 二维码场景适配

| 需求 | 满足度 |
|---|---|
| 固定长度，适合小尺寸打印 | 8 字符，Version 1 QR 码（21x21 模块，25 字母数字） |
| 非连接场景可读取 | 短码扫描成功率高 |
| 无隐私数据泄露 | 码值不含日期、流水号等有意义信息 |
| 可撤销 | 数据库 soft delete + 标记已用 |

**建议**：二维码内容直接使用 8 位码值，不附加 URL 前缀。由扫描端应用层决定如何路由。

### 6.3 资产编号场景适配

**不推荐使用**。

| 需求 | 8 位 Base31 码 | PRD 格式 `ASSET+日期+流水` |
|---|---|---|
| 按日期排序 | 不可，随机顺序 | 可，日期前缀天然有序 |
| 生成时间可视化 | 不可 | 可，日期段直接可读 |
| 大规模无碰撞 | 概率性（>500K 风险） | 确定性（流水号保证） |
| 固定长度 | 是 | 否 |

**建议**：资产编号继续使用 PRD 定义的 `ASSET+日期+流水` 格式。8 位 Base31 码作为**二维码令牌**嵌入标签，通过映射表关联到资产记录。

---

## 7. 代码包结构

```
js.api-wms/pkg/token/
├── doc.go              // 包文档
├── base31.go           // Base31 编码/解码实现
├── base31_test.go      // 编码往返测试、边界测试
├── generator.go        // Token 生成器、种子构造
├── generator_test.go   // 唯一性测试、确定性测试
└── benchmark_test.go   // 性能基准：单体生成、并发 100 万生成
```

### 7.1 接口设计

```go
package token

// Encode 将 uint64 编码为 8 位 Base31 字符串。
// 左填充为 '2'（字符集中第 0 个字符）。
func Encode(n uint64) string

// Decode 将 8 位 Base31 字符串解码为 uint64。
// 返回错误如果字符不在合法字符集中。
func Decode(s string) (uint64, error)

// Generate 生成一个随机唯一令牌。
func Generate() string

// MustGenerate 生成令牌，若随机源不可用则 panic。
func MustGenerate() string
```

### 7.2 Base31 编码实现要点

```go
const (
    base      = 31
    chars     = "23456789ABCDEFGHJKMNPQRSTUVWXYZ"
    tokenLen  = 8
    bitMask   = 0x7F_FFFF_FFFF // 39 bits
)

var decodeMap [256]byte // 初始化时填充，ASCII -> 值映射

func init() {
    for i := range decodeMap {
        decodeMap[i] = 0xFF // 非法字符标记
    }
    for i, c := range []byte(chars) {
        decodeMap[c] = byte(i)
    }
}
```

### 7.3 生成器实现要点

```go
var processStart = uint64(runtime.Nanotime()) // 包初始化时记录

func Generate() string {
    mono := uint64(runtime.Nanotime()) - processStart
    var randBytes [8]byte
    _, err := crypto.Read(randBytes[:])
    if err != nil {
        // 降级：使用 runtime.fastrand64() 作为最后防线
        mono ^= runtime.FastRand64()
    } else {
        mono ^= binary.LittleEndian.Uint64(randBytes[:])
    }
    seed := makeSeed(mono)
    hash := cityhash.CityHash64(seed, uint32(len(seed)))
    return Encode(hash & bitMask)
}
```

### 7.4 测试要点

| 测试 | 目标 |
|---|---|
| `TestEncodeDecode` | 往返编码一致性（0, 1, 2^39-1, 随机值 x 1000） |
| `TestEncodePadding` | 小数值左填充至 8 位 |
| `TestDecodeInvalidChar` | 非法字符错误处理 |
| `TestGenerate` | 连续 10 万生成无碰撞 |
| `TestConcurrentGenerate` | 100 goroutine x 10,000 = 100 万生成无碰撞 |
| `BenchmarkGenerate` | 单线程生成吞吐量 |
| `BenchmarkGenerateParallel` | 多线程生成吞吐量 |
| `BenchmarkEncode` | 编码微基准 |

### 7.5 Benchmark 预期

| 基准 | 目标值 |
|---|---|
| `BenchmarkGenerate-8` | >=200,000 ns/op |
| `BenchmarkGenerateParallel-8` | >=1,000,000 ops/s |
| `BenchmarkEncode-8` | <=20 ns/op |
| 并发 100 万测试 | <=30 s (with -race) |
| 内存分配 | 0 B/op, 0 allocs/op（所有缓冲区栈分配） |

---

## 8. 风险汇总

| 风险 | 等级 | 缓解措施 |
|---|---|---|
| 39 bit 容量上限（~55B 最大值） | **高** | 文档明确容量限制；日志告警阈值 500K；>1M 拒绝使用 |
| 无校验位，人工输入无防错 | 中 | 确认设计决策：8 位不加长。依赖数据库查询验证 |
| `crypto/rand` 在容器环境可能阻塞 | 低 | 降级方案：`runtime.FastRand64()` |
| 无时间排序能力（不适用资产编号） | 低 | 明确场景分离：资产编号用 `ASSET+日期+流水` |
| CityHash 确定性：同种子->同码 | 低 | 种子含真随机分量，实际不可行触发 |
| 随机 ID 写入导致数据库页分裂 | 中 | <500K 记录可接受；超量建议时间前缀设计 |

---

## 9. 演进路线

| 阶段 | 内容 |
|---|---|
| Phase 1（当前） | 实现基础生成器 + 内存去重测试 |
| Phase 2 | 数据库集成：UNIQUE 约束 + INSERT 重试 |
| Phase 3 | 可选：生成器缓冲池优化 `crypto/rand` 调用 |
| Phase 4 | 可选：9 位扩展方案（如需 >1M 码容量） |

---

## 附录 A：字符集对比

```
            1 2 3 4 5 6 7 8 9 0 A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
Base36      1 2 3 4 5 6 7 8 9 0 A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
Crockford   1 2 3 4 5 6 7 8 9 0 A B C D E F G H - J K L M N - P Q R S T - V W X Y Z
Base31      - 2 3 4 5 6 7 8 9 - A B C D E F G H - J K - M N - P Q R S T U V W X Y Z
```

Crockford 排除 `I/L/O` 但保留 `0/1`；Base31 额外排除 `0/1`，仅含无混淆字符。

## 附录 B：为何不用 ULID / UUIDv7

| 方案 | 长度 | 排序 | 可读性 | 适用性 |
|---|---|---|---|---|
| UUIDv4 | 36 chars | 否 | 差 | 不适合人工/二维码 |
| UUIDv7 | 36 chars | 时间排序 | 差 | 同上 |
| ULID | 26 chars | 时间排序 | 中 | 过长，对打印标签不友好 |
| NanoID (21) | 21 chars | 否 | 中 | 仍偏长 |
| **Base31 (8)** | **8 chars** | 否 | 优 | 专为二维码/人工优化 |
