+++
title= "10位短码方案设计"
description= "10位短码设计"
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

**版本** 1.0
**日期** 2026-07-07


---

## 1. 设计目标

| 目标 | 说明 |
|---|---|
| **10 位固定长度** | 短小精悍，适合标签打印和二维码嵌入 |
| **第1位** | 首位，对当前年份与 31 取余数，利于数据库索引和减少碰撞概率 |
| **第2–9位** | 8 位随机负载，39 bit 编码输入，提供唯一性核心 |
| **第10位** | 末位，对前 9 个字符进行 mod-31 求和校验，尽早拦截输入错误 |
| **Base31 编码** | 排除 `0/O/1/I/L`，零混淆 |
| **二维码友好** | 10 字符，Version 1 QR 可容纳（最大 25 字母数字） |
| **人工易输入** | 无大小写歧义，无数字字母混淆 |
| **概率唯一** | 种子哈希 + 年份分桶 + 数据库 UNIQUE 约束三道防线 |

### 适用场景

- **二维码令牌**：嵌入二维码，扫描后查找对应业务实体
- **短期操作凭证**：入库确认、验收签收等一次性操作
- **不适用场景**：资产编号（推荐使用 PRD 定义的 `ASSET+日期+流水`）、合同编号等需要精确日期可读性的场景

### 格式示意

```
┌──────────┬──────────────────────────┬──────────┐
│  Pos 1   │        Pos 2–9           │  Pos 10  │
│ 年份前缀  │      随机负载 (8 chars)    │  校验码   │
│year % 31 │      39 bit 输入          │ mod-31 和│
└──────────┴──────────────────────────┴──────────┘
```

**示例**：年份 2026 → `2026 % 31 = 12` → 字符 `D`

```
D 8 K 3 D 7 P W N T
│ └── 8 chars random ──┘ │
│               checksum ─┘
year prefix (12 -> 'D')
```

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

### 2.2 字符 → 索引映射表

| 索引 | 字符 | 索引 | 字符 | 索引 | 字符 |
|---|---|---|---|---|---|
| 0 | 2 | 11 | D | 22 | R |
| 1 | 3 | 12 | E | 23 | S |
| 2 | 4 | 13 | F | 24 | T |
| 3 | 5 | 14 | G | 25 | U |
| 4 | 6 | 15 | H | 26 | V |
| 5 | 7 | 16 | J | 27 | W |
| 6 | 8 | 17 | K | 28 | X |
| 7 | 9 | 18 | M | 29 | Y |
| 8 | A | 19 | N | 30 | Z |
| 9 | B | 20 | P | | |
| 10 | C | 21 | Q | | |

### 2.3 为什么选择 31 个字符

| 基数 | 8 位容量（有效比特） | 剔除易混淆字符 | 评价 |
|---|---|---|---|
| Base36 | 41.4 bit | 否 | 包含 `0/O/1/I/L`，人工输入易错 |
| Base32 (RFC 4648) | 40.0 bit | 否（含 `O/0/1/L`） | 标准编码，但字符集对人工不友好 |
| Base32 (Crockford) | 40.0 bit | 部分（无 `I/L/O`，但有 `0/1`） | 接近理想，含数字 0/1 仍有混淆风险 |
| **Base31** | **39.6 bit** | **完全** | 所有易混淆字符全部剔除 |

31 是剔除 5 个易混淆字符后自然剩余的数量。容量损失极小（Base32 → Base31 仅损失 ~0.4 bit），换取了零歧义的人工输入体验。

### 2.4 容量计算

| 分量 | 公式 | 值 |
|---|---|---|
| 总编码空间 | 31^10 | ~8.20 × 10^14 |
| 年份前缀占用 | 1 char，31 有效值 | 31 桶 |
| 随机负载空间 | 31^8 (Pos 2–9) | ~8.53 × 10^12 = 39.63 bit |
| 校验码占用 | 1 char（确定性） | 0 额外容量 |
| **有效随机位数** | **39 bit** | ~5.50 × 10^11 个值 |

注意：年份前缀和校验码**不增加随机容量**，它们是结构性字段。核心随机空间仍为 39 bit。10 位方案相比 8 位的增益在于**结构性防护**（年份分桶 + 校验码）而非容量扩张。

---

## 3. 编码算法设计

### 3.1 Generator 算法流程

```
输入: 年份 year (e.g., 2026)
输出: token string (10 字符)

Step 0 — 年份前缀
    yearIdx  = year % 31
    yearChar = base31Chars[yearIdx]             // 2026 → 'D'

Step 1 — 构造种子
    mono  = runtime.nanotime() - processStart   // 单调纳秒
    rand  = crypto.Read([8]byte)                // 64 bit 真随机
    seed  = binary.LittleEndian.Uint64(rand) ^ uint64(mono)

Step 2 — 哈希打散
    hash  = CityHash64(seed)

Step 3 — 截取 & 编码负载
    bits    = hash & 0x7F_FFFF_FFFF             // 低 39 bit
    payload = Base31Encode(bits)                // 8 字符,左填充 '2'

Step 4 — 计算校验码
    sum = 0
    for c in (yearChar + payload):              // 遍历前 9 字符
        sum += base31DecodeMap[c]
    checksumIdx  = sum % 31
    checksumChar = base31Chars[checksumIdx]

Step 5 — 组装
    token = yearChar + payload + checksumChar   // 1 + 8 + 1 = 10
```

### 3.2 Seed 组成详解

| 组件 | 来源 | 熵 (bits) | 说明 |
|---|---|---|---|
| monotonic nanos | `runtime.nanotime()` - 进程启动时间 | ~0.03/ns | 单调递增，不受 NTP 时钟跳变影响，防止时钟回拨导致种子重复 |
| crypto/rand | CSPRNG | 64 | 真随机性来源，保证多实例并行生成时无冲突 |
| year | 传入参数 | 0 | 确定性参数，不计入熵。作为输出结构的一部分嵌入 |

相较于原 8 位方案，移除了 UTC 日期。理由：
- UTC 日期的精度为 1 天，对同日调用贡献 0 额外熵
- monotonic nanos 已提供更细粒度的时序信息
- year 作为确定性前缀嵌入码值，不参与随机种子构造

### 3.3 CityHash64 选择理由

| 属性 | CityHash64 | SipHash | XXH64 |
|---|---|---|---|
| 雪崩效应 | 优秀 — 1 bit 输入翻转导致 ~32 bit 输出翻转 | 同样优秀 | 短输入较弱 |
| 吞吐量（单核） | ~1.5 GB/s | ~0.5 GB/s | ~2.5 GB/s |
| Go 实现 | ~300 行纯 Go，无 CGO | ~370 行 | 需 CGO 或质量不确定的端口 |
| 安全性 | 非加密哈希 | 加密级哈希 | 非加密哈希 |
| 结论 | **选择** | 过度（加密级，慢 3×） | 不选（短输入分布不匀） |

**为什么不选 SipHash**：SipHash 设计目标是防 Hash DoS 攻击（带密钥），速度慢约 3 倍。本方案不需要加密强度哈希，CityHash 的速度优势更关键。

**为什么不选 XXH64**：XXH64 在单字节输入下雪崩效应较弱。本方案的种子仅 16 字节，CityHash 在短输入上表现更稳定。

### 3.4 Base31 编码器实现要点

```go
const base31Chars = "23456789ABCDEFGHJKMNPQRSTUVWXYZ"
const payloadLen = 8
const bitMask = 0x7F_FFFF_FFFF // 39 bits

var decodeMap [256]byte

func init() {
    for i := range decodeMap {
        decodeMap[i] = 0xFF
    }
    for i := 0; i < len(base31Chars); i++ {
        decodeMap[base31Chars[i]] = byte(i)
    }
}

func Encode(n uint64) string {
    if n == 0 {
        return "22222222"
    }
    var buf [payloadLen]byte
    i := payloadLen - 1
    for n > 0 && i >= 0 {
        buf[i] = base31Chars[n%31]
        n /= 31
        i--
    }
    for j := 0; j <= i; j++ {
        buf[j] = '2' // 左填充，'2' = 索引 0
    }
    return string(buf[:])
}

func Decode(s string) (uint64, error) {
    if len(s) != payloadLen {
        return 0, fmt.Errorf("expected %d chars, got %d", payloadLen, len(s))
    }
    var n uint64
    for i := 0; i < payloadLen; i++ {
        v := decodeMap[s[i]]
        if v == 0xFF {
            return 0, fmt.Errorf("invalid char '%c' at position %d", s[i], i)
        }
        n = n*31 + uint64(v)
    }
    return n, nil
}
```

### 3.5 校验码算法

**算法**：前 9 个字符的索引值求和，取模 31。

```
checksum = (index(yearChar) + sum(index(payload[i])) for i in 0..7) % 31
```

**检测能力**：

| 错误类型 | 检测率 | 说明 |
|---|---|---|
| 单字符替换（A → B） | 100% | 差值不为 31 的倍数时为 100% 检测。在索引值 0–30 范围内，差值不可能为 31 的倍数 |
| 相邻字符对调（AB → BA） | 0% | 简单求和无法检测顺序变化 |
| 任意双字符错误 | ~97% | 1 - 1/31 |
| 长度错误 | 100% | 长度校验独立于 checksum |

**限制说明**：简单 mod-31 求和无法检测字符顺序调换。如需检测相邻对调，可升级为加权校验 `sum((i+1) * value[i]) % 31`。当前选择简单求和，满足基本需求。

### 3.6 校验器实现要点

```go
const TokenLen = 10

func Validate(token string) bool {
    if len(token) != TokenLen {
        return false
    }
    sum := 0
    for i := 0; i < TokenLen-1; i++ {
        v := decodeMap[token[i]]
        if v == 0xFF {
            return false
        }
        sum += int(v)
    }
    expected := decodeMap[token[TokenLen-1]]
    if expected == 0xFF {
        return false
    }
    return sum%31 == int(expected)
}
```

### 3.7 Generator 实现要点

```go
var processStart = uint64(runtime.Nanotime())

func Generate(year int) string {
    // Step 0: 年份前缀
    yearIdx := year % 31
    yearChar := base31Chars[yearIdx]

    // Step 1: 构造种子
    mono := uint64(runtime.Nanotime()) - processStart
    var rnd [8]byte
    if _, err := crypto.Read(rnd[:]); err != nil {
        mono ^= runtime.FastRand64()
    } else {
        mono ^= binary.LittleEndian.Uint64(rnd[:])
    }

    // Step 2-3: 哈希 + 编码
    hash := cityhash.CityHash64(mono)
    payload := Encode(hash & bitMask)

    // Step 4: 校验码
    sum := int(decodeMap[yearChar])
    for i := 0; i < len(payload); i++ {
        sum += int(decodeMap[payload[i]])
    }
    checksumChar := base31Chars[sum%31]

    // Step 5: 组装
    return string(yearChar) + payload + string(checksumChar)
}
```

---

## 4. 唯一性分析

### 4.1 年份分桶机制

年份前缀 `year % 31` 将码空间分为 31 个桶。不同年份桶之间的码**永远不会碰撞**（前缀不同）。

示例：
- 2026 年 → `2026 % 31 = 12` → 前缀 `D`，码值 `D·········`
- 2027 年 → `2027 % 31 = 13` → 前缀 `E`，码值 `E·········`
- `D` 和 `E` 前缀的码天然不冲突

**31 年回环**：2057 年 `2057 % 31 = 12` → 前缀再次为 `D`，与 2026 年共享前缀桶。但在实际 WMS 业务中，31 年前的资产码索引已退役，此冲突无实际影响。

### 4.2 单年桶内碰撞概率

每桶内随机分量为 39 bit（2^39 ≈ 5.50 × 10^11 个不同值）。

基于生日悖论，单年桶内生成 N 个码后至少出现一次碰撞的概率：

```
P(collision) ≈ 1 - exp(-N² / (2 × 2^39))
```

| 单年生成 N | 碰撞概率 | 平均每插入重试次数 |
|---|---|---|
| 1,000 | ~0.00009% | ~1.0 |
| 10,000 | ~0.009% | ~1.0 |
| 100,000 | ~0.9% | ~1.01 |
| 500,000 | ~20% | ~1.25 |
| **740,000** | **~50%** | ~2.0 |
| 1,000,000 | ~60% | ~2.5 |
| 2,000,000 | ~97% | ~33 |
| 5,000,000 | ~99.9999% | 不可用 |

### 4.3 容量规划建议

| 单年规模 | 建议 |
|---|---|
| < 100,000 码/年 | 直接使用，无需额外措施 |
| 100,000 ~ 500,000 码/年 | 监控碰撞率 |
| 500,000 ~ 1,000,000 码/年 | 日志告警，考虑扩展 |
| > 1,000,000 码/年 | 不推荐。升级为更多随机位（如改为 9 位随机负载） |

### 4.4 与 8 位方案的碰撞对比

| 场景 | 8 位方案（全局空间） | 10 位方案（年桶隔离） |
|---|---|---|
| 100K/年 × 5 年 | 500K 全局 → 20% 碰撞 | 100K/年/桶 → ~0.9%/年 |
| 100K/年 × 10 年 | 1M 全局 → 60% 碰撞 | 100K/年/桶 → ~0.9%/年（跨年零碰撞） |
| P50 碰撞点 | 740K 全局 | 740K/单年桶 |
| **实际可用年限** | 5–7 年 | >31 年（回环前） |

**结论**：年份分桶将"全局累积"转化为"年度累积"，使方案在 100K 码/年的典型 WMS 场景下可用期延长至 31 年以上。

### 4.5 数据库 UNIQUE 约束

CityHash64 是确定性函数：相同的种子输入产生相同的输出。可能产生相同码的场景：

1. 两个不同种子经 CityHash64 后低 39 bit 相同 → 概率性碰撞
2. `crypto/rand` 调用失败且 monotonic nanos 相同 → 极度罕见
3. 同种子同 CityHash 输出 → 确定性（预期行为）

数据库 `UNIQUE` 约束作为最后防线：

```sql
INSERT INTO tokens (code) VALUES ($1)
ON CONFLICT (code) DO NOTHING RETURNING id
```

受影响的航数为 0 时，重新生成（最多重试 10 次）。10 次重试后冲突则返回明确错误。

### 4.6 重试策略

```
// Layer 1: 内存去重（测试/benchmark 环境）
if inMemorySet.exists(token):
    retry++

// Layer 2: 校验码验证（内存快速过滤）
if !Validate(token):
    // 不会因生成逻辑产生，但作为防御性检查
    retry++

// Layer 3: 数据库 UNIQUE 约束（生产环境）
INSERT ... ON CONFLICT DO NOTHING RETURNING id
if no rows returned:
    retry++
```

---

## 5. 性能分析

### 5.1 单次生成耗时

| 步骤 | 耗时 (µs) | 占比 |
|---|---|---|
| `runtime.nanotime()` | ~0.02 | <1% |
| `crypto/rand` (8 bytes) | 1–5 | 55–80% |
| CityHash64 | ~0.05 | ~1% |
| Base31 编码 (8 chars) | ~0.1 | ~2% |
| 校验码计算 (9 char 求和) | ~0.05 | ~1% |
| 组装字符串 | ~0.1 | ~2% |
| **总计（纯生成）** | **~2–6** | — |
| 数据库 INSERT + UNIQUE | ~1000 | DB 瓶颈 |

与 8 位方案相比，额外的校验码计算开销可忽略（+0.05 µs）。

### 5.2 吞吐量估算

| 场景 | 吞吐量 | 瓶颈 |
|---|---|---|
| 纯生成（无冲突检查） | ~170K–500K/s 单核 | `crypto/rand` 速率 |
| 生成 + 内存去重（N=100K） | ~200K/s 单核 | ~1.01 次平均重试 |
| 生成 + DB 写入 | ~1,000/s 单连接 | 数据库 I/O |
| 生成 + DB + 连接池（10 连接） | ~8,000/s | 数据库并发能力 |

### 5.3 Validate() 性能

| 场景 | 耗时 |
|---|---|
| 合法码（完整验证） | ~30 ns |
| 长度错误（早期返回） | ~2 ns |
| 非法字符（首字符检测） | ~20 ns |
| 批量验证 100 万码 | ~30 ms |

### 5.4 多实例部署

多实例并行生成时碰撞概率增加，但分布式特性不会引入新风险：

- 各实例独立调用 `crypto/rand`，种子天然不相关
- 年份前缀由 year 参数统一决定（所有实例生成同年码时前缀一致）
- 碰撞概率仅取决于总生成数量，与实例数目无关
- 数据库 UNIQUE 约束跨实例统一生效

| 实例数 | 每实例码数 | 总码数 | 单年桶碰撞概率 |
|---|---|---|---|
| 1 | 100,000 | 100,000 | ~0.9% |
| 2 | 100,000 | 200,000 | ~3.5% |
| 5 | 100,000 | 500,000 | ~20% |
| 10 | 50,000 | 500,000 | ~20% |

### 5.5 优化建议

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

预读取放大因子 8×，将 `crypto/rand` 调用次数压缩至 1/8。

---

## 6. 安全分析

### 6.1 可预测性

| 攻击向量 | 难度 | 说明 |
|---|---|---|
| 从已观察码值反推种子 | 不可行 | CityHash 是单射函数，无法逆推；即使反推出种子，`crypto/rand` 每个种子都不同 |
| 暴力枚举 2^39 空间 | ~17 年/1000 QPS | 实际不可行，但空间只有 39 bit，不抗超级计算机 |
| 生日攻击（寻找碰撞） | O(2^(39/2)) = O(2^19.5) = 740K 次尝试 | 如有已知码，可理论寻找碰撞。但攻击者无利可图 — 无会话劫持、权限提升等价值 |
| 时序攻击（猜测下一码） | 不可行 | 每码独立随机生成，无序列关系 |
| 年份前缀泄露时间信息 | 低风险 | 可推断码的大致年份（±15 年模糊度，因 mod 30 回环）。每个码值泄露 ~4.9 bit 时间信息。WMS 资产年份非敏感信息 |

**结论**：39 bit 空间不满足加密安全（cryptographic security）要求（通常需要 ≥128 bit），但对于 WMS 二维码令牌场景完全足够。攻击者没有经济利益动机进行碰撞攻击。

### 6.2 二维码场景适配

| 需求 | 满足度 |
|---|---|
| 固定长度，适合小尺寸打印 | 10 字符，Version 1 QR 码（21×21 模块，最大 25 字母数字） |
| 非连接场景可读取 | 短码扫描成功率高 |
| 无隐私数据泄露 | 码值不含日期、流水号等有意义信息（除年份前缀的 ±15 年模糊度） |
| 可撤销 | 数据库 soft delete + 标记已用 |
| 输入错误检测 | 内置校验码，即刻拒绝 100% 单字符错误 |

**建议**：二维码内容直接使用 10 位码值，不附加 URL 前缀。由扫描端应用层决定如何路由。

### 6.3 资产编号场景适配

**不推荐使用**。

| 需求 | 10 位 Base31 码 | PRD 格式 `ASSET+日期+流水` |
|---|---|---|
| 按日期排序 | 可（年份桶内无内部排序） | 可（精确日期排序） |
| 人员可读日期 | 模糊（year % 31 → ±15 年模糊度） | 精确（年月日明确） |
| 大规模无碰撞 | 概率性（>500K/年风险） | 确定性（流水号保证） |
| 固定长度 | 10 字符 | 否 |
| 错误检测 | 内置校验码 | 无 |

**建议**：资产编号继续使用 PRD 定义的 `ASSET+日期+流水` 格式。10 位 Base31 码作为**二维码令牌**嵌入标签，通过映射表关联资产记录。

### 6.4 校验码的安全防护价值

| 场景 | 无校验码（8 位方案） | 有校验码（10 位方案） |
|---|---|---|
| 人工输入错误 1 字符 | 需 DB 查询才知无效 | 前端 ~30ns 即可判断 |
| 扫描错误后手动校正 | 无提示 | 校验码不匹配 → 要求重输 |
| 批量导入逐条验证 | 需 N 次 DB 查询 | 内存 O(n) 批量校验 |
| 防伪造码（非生成器产出） | 无 | 随机猜测 10 位合法码概率 = 1/31 ≈ 3.2% |

**实际收益**：对于移动端手动输入码值的回退场景，校验码可立即拒绝 ≥97% 的输入错误，避免无效 DB 查询。这是 10 位方案相比 8 位方案的**最大实际增益**。

---

## 7. 代码包结构

```
js.api-wms/pkg/token/
├── doc.go               // 包文档
├── base31.go            // Base31 编码/解码实现 + decodeMap
├── base31_test.go       // 编码往返测试、填充测试、非法字符测试
├── generator.go         // Generate(year) + Validate(token) + ExtractPayload
├── generator_test.go    // 唯一性测试、校验码正确性测试、并发测试
└── benchmark_test.go    // 性能基准：生成、编码、校验
```

### 7.1 接口设计

```go
package token

const (
    TokenLen   = 10   // 完整码长度
    PayloadLen = 8    // 随机负载长度
    Base       = 31   // 编码基数
    Chars      = "23456789ABCDEFGHJKMNPQRSTUVWXYZ"
    BitMask    = 0x7F_FFFF_FFFF // 39 bits
)

// Encode 将 uint64 编码为 8 位 Base31 字符串（负载部分）。
// 左填充为 '2'（字符集中索引 0 的字符）。
func Encode(n uint64) string

// Decode 将 8 位 Base31 字符串解码为 uint64。
// 返回错误如果字符不在合法字符集中。
func Decode(s string) (uint64, error)

// Generate 生成一个带指定年份前缀和校验码的 10 位令牌。
func Generate(year int) string

// MustGenerate 同 Generate，随机源不可用时 panic。
func MustGenerate(year int) string

// Validate 验证 10 位令牌的校验码是否正确。
// 在访问数据库前使用，快速拦截输入错误。
func Validate(token string) bool

// ExtractPayload 从 10 位令牌中提取 8 位随机负载部分。
func ExtractPayload(token string) (string, error)
```

### 7.2 测试要点

| 测试 | 目标 |
|---|---|
| `TestEncodeDecodeRoundtrip` | 往返编码一致性：0, 1, 2^39-1, 1000 个随机值 |
| `TestEncodePadding` | 小输入值左填充至 8 位 |
| `TestDecodeInvalidChar` | 非法字符错误处理 |
| `TestDecodeInvalidLength` | 非 8 位输入错误处理 |
| `TestGenerateFormat` | 输出长度为 10，前 9 字符校验和 = 第 10 字符 |
| `TestValidate` | 合法码通过、单字符错误不通过、双字符错误不通过、长度错误不通过 |
| `TestGenerateYearPrefix` | year 2026 → 前缀 'D', year 2027 → 前缀 'E', year 2057 → 前缀 'D' (同余) |
| `TestGenerateNoCollision` | 连续 50,000 生成无碰撞（单桶内） |
| `TestConcurrentGenerate` | 100 goroutine × 10,000 = 100 万生成无碰撞（单桶内） |
| `TestValidateRejectsSwap` | 对调相邻字符 → 期望通过（基于简单求和的已知限制） |
| `BenchmarkGenerate` | 单线程生成吞吐量 |
| `BenchmarkGenerateParallel` | 并发生成吞吐量 |
| `BenchmarkEncode` | 编码微基准 |
| `BenchmarkValidate` | 校验微基准（合法/非法） |

### 7.3 Benchmark 预期

| 基准 | 目标值 |
|---|---|
| `BenchmarkGenerate-8` | ≤5 µs/op |
| `BenchmarkGenerateParallel-8` | ≥500,000 ops/s |
| `BenchmarkEncode-8` | ≤20 ns/op |
| `BenchmarkValidate-8` | ≤30 ns/op |
| 并发 100 万生成测试 | ≤30 秒 (with -race) |
| 内存分配 | 0 B/op, 0 allocs/op（所有缓冲区栈分配） |

---

## 8. 风险汇总

| 风险 | 等级 | 缓解措施 |
|---|---|---|
| **39 bit 容量上限（单桶 ~55B）** | 高 | 年份分桶将碰撞限制在单年内；单年 >500K 日志告警；>1M 拒绝使用 |
| **年份前缀 31 年回环** | 低 | 2057 年回环前退役旧码 |
| **校验码无法检测字符对调** | 低 | 简单求和的已知限制；如需检测对调，可升级为加权校验（1 行代码改动） |
| **无精确时间可读性** | 低 | 设计权衡：短码优先于信息密度。年份前缀提供模糊时间可见性 |
| `crypto/rand` 在容器环境可能阻塞 | 低 | 降级方案：`runtime.FastRand64()` |
| 随机 ID 写入导致数据库页分裂 | 中 | 年份前缀提供粗粒度 B-tree 局部性（同年前缀码聚集在同一区域）；<500K 记录可接受 |
| 年份前缀泄露资产大致年龄 | 低 | WMS 资产年份非敏感信息；如需隐藏，可改用随机前缀（代价：丢失 DB 索引局部性） |
| 无校验位导致输入错误延迟发现 | — | **已解决**：10 位方案内置校验码，前端即刻验证 |

---

## 9. 演进路线

| 阶段 | 内容 |
|---|---|
| Phase 1（当前） | 实现 Base31 编码器 + Generator + Validator + 内存去重测试 |
| Phase 2 | 数据库集成：UNIQUE 约束 + INSERT ON CONFLICT 重试 |
| Phase 3 | 可选：加权校验升级（如业务需要检测对调错误） |
| Phase 4 | 可选：9 位随机负载扩展方案（如单年 >1M 码需求，提供 ~49 bit 随机空间） |

---

## 附录 A：8 位 vs 10 位方案对比

| 维度 | 8 位方案 | 10 位方案 |
|---|---|---|
| 长度 | 8 chars | 10 chars |
| 有效随机位 | 39 bit | 39 bit（相同） |
| 年份可见性 | 无 | `year % 31`（模糊 ±15 年） |
| 错误检测 | 无 | 校验码（100% 单字符替换检测） |
| DB 索引局部性 | 无（完全随机） | 有（同年前缀聚集） |
| 单年碰撞隔离 | 无（全局累积） | 有（年桶隔离） |
| QR 密度 | Version 1 (21×21) | Version 1 (21×21) — 均低于 25 char 限制 |
| 可用年限（100K 码/年） | ~5–7 年 | >31 年 |
| 核心选择理由 | 最短长度 | 最长可用寿命 + 输入防护 |

## 附录 B：校验码计算示例

```
给定 token = "D8K3D7PWNT"
前 9 字符：D, 8, K, 3, D, 7, P, W, N
索引值：   11, 6, 17, 1, 11, 5, 20, 27, 19

sum = 11+6+17+1+11+5+20+27+19 = 117
117 % 31 = 24 → 索引 24 = char 'T'

验证时：重新计算前 9 位 sum % 31，与第 10 位字符索引比较
若结果匹配 → token 合法
```

## 附录 C：为何不用 ULID / UUIDv7

| 方案 | 长度 | 排序 | 可读性 | 适用性 |
|---|---|---|---|---|
| UUIDv4 | 36 chars | 否 | 差 | 不适合人工/二维码 |
| UUIDv7 | 36 chars | 时间排序 | 差 | 同上 |
| ULID | 26 chars | 时间排序 | 中 | 过长，对打印标签不友好 |
| NanoID (21) | 21 chars | 否 | 中 | 仍偏长 |
| **Base31 (10)** | **10 chars** | 年份桶内 | 优 | 专为二维码 + 校验 + DB 局部性优化 |
| **Base31 (8)** | **8 chars** | 否 | 优 | 最短长度优化 |

## 附录 D：JavaScript 校验算法

用于 HarmonyOS 客户端 (`js.terminal`) 和 Web 前端的扫码输入校验，在请求后端 API 之前快速拦截格式错误和输入错误。

### D.1 字符集与解码映射

```javascript
const BASE31_CHARS = "23456789ABCDEFGHJKMNPQRSTUVWXYZ";

/** @type {Object<string, number>} */
const DECODE_MAP = Object.freeze({
  '2': 0,  '3': 1,  '4': 2,  '5': 3,  '6': 4,  '7': 5,  '8': 6,  '9': 7,
  'A': 8,  'B': 9,  'C': 10, 'D': 11, 'E': 12, 'F': 13, 'G': 14, 'H': 15,
  'J': 16, 'K': 17, 'M': 18, 'N': 19, 'P': 20, 'Q': 21, 'R': 22, 'S': 23,
  'T': 24, 'U': 25, 'V': 26, 'W': 27, 'X': 28, 'Y': 29, 'Z': 30
});
```

### D.2 校验函数（返回详细结果）

```javascript
/**
 * 验证 10 位 Base31 令牌的校验码是否正确。
 * 在请求后端 API 之前使用，立即拦截输入错误。
 *
 * @param {string} token - 10 位令牌字符串
 * @returns {{ valid: boolean, reason?: string }}
 */
function validateToken(token) {
  if (!token || token.length !== 10) {
    return { valid: false, reason: "长度必须为10位" };
  }

  let sum = 0;
  for (let i = 0; i < 9; i++) {
    const ch = token[i];
    const v = DECODE_MAP[ch];
    if (v === undefined) {
      return { valid: false, reason: "第" + (i + 1) + "位字符 '" + ch + "' 不合法" };
    }
    sum += v;
  }

  const checksumChar = token[9];
  const expected = DECODE_MAP[checksumChar];
  if (expected === undefined) {
    return { valid: false, reason: "校验位字符 '" + checksumChar + "' 不合法" };
  }

  if (sum % 31 !== expected) {
    return { valid: false, reason: "校验码不匹配，请检查输入" };
  }

  return { valid: true };
}
```

### D.3 快速校验版（返回 boolean）

```javascript
/**
 * 快速校验，适合高频调用场景（如扫码回调、表单 oninput）。
 *
 * @param {string} token
 * @returns {boolean}
 */
function isValidToken(token) {
  if (token == null || token.length !== 10) return false;
  let sum = 0;
  for (let i = 0; i < 9; i++) {
    const v = DECODE_MAP[token[i]];
    if (v === undefined) return false;
    sum += v;
  }
  const expected = DECODE_MAP[token[9]];
  if (expected === undefined) return false;
  return sum % 31 === expected;
}
```

### D.4 使用示例

```javascript
// 合法令牌
validateToken("D8K3D7PWNT");
// → { valid: true }

isValidToken("D8K3D7PWNT");
// → true

// 单字符输入错误
validateToken("D8K3D7PWNX");
// → { valid: false, reason: "校验码不匹配，请检查输入" }

// 长度错误
validateToken("D8K3D7P");
// → { valid: false, reason: "长度必须为10位" }

// 非法字符（含被排除的字符 '0', '1', 'O', 'I', 'L'）
validateToken("D8K3D7PN01");
// → { valid: false, reason: "第9位字符 '0' 不合法" }
```

### D.5 Go 与 JS 互操作验证

两种语言基于相同的 Base31 字符集和 mod-31 求和规则，同一令牌在两端的校验结果一致：

```
token:  D8K3D7PWNT

Go:
  sum = 11+6+17+1+11+5+20+27+19 = 117
  117 % 31 = 24 → char[24] = 'T'  → 通过

JS:
  sum = 11+6+17+1+11+5+20+27+19 = 117
  117 % 31 = 24 → char[24] = 'T'  → 通过

两者结果一致 ✓
```

### D.6 HarmonyOS 扫码端集成示例

在 `js.terminal` 扫码回调中使用：

```javascript
// 扫码回调
function onQRCodeScanned(code) {
  const result = validateToken(code);
  if (!result.valid) {
    // 前端立即拒绝，不发起网络请求
    vibrate();
    showToast(result.reason);
    return;
  }

  // 校验通过，调用后端 API 查询令牌对应的业务实体
  fetch('/api/v1/token/lookup', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ token: code })
  })
    .then(res => res.json())
    .then(data => handleLookupResult(data))
    .catch(err => handleNetworkError(err));
}

// 手动输入时的实时校验
inputField.addEventListener('input', (e) => {
  const value = e.target.value.toUpperCase();
  if (value.length === 10) {
    const ok = isValidToken(value);
    inputField.classList.toggle('invalid', !ok);
  }
});
```
