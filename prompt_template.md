# 提示词模板与干扰段落生成函数（伪代码）

本文件描述了用于"Twin LLM token 上限探针"的提示词构造方法。所有探针调用都基于同一个**结构相同、长度不同**的中英混合 prompt 模板，靠改变干扰段落数量来控制总字符数。

## 一、整体结构

```text
[HEADER 任务说明 + TypeScript 代码]
  ↓
[FILLER#1] [FILLER#2] ... [FILLER#N]    ← 长度由 N 控制
  ↓
[TAIL 重复约束 + 复述要求]
```

## 二、HEADER（固定）

```text
你是一名资深分布式系统工程师。请严格阅读下面所有"FILLER 段落"，每段都有形如【FILLER#N】的编号。你必须找出最大的 N（即最后一段的编号）。

# 任务
1) 写一段中文 summary（至少 200 个汉字），最后一句必须是「最后干扰段落编号为 X」，X 为最后 FILLER 段落的实际数字。
2) 对下面这段 TypeScript 代码做 code_review，至少 4 条中文意见：
\`\`\`ts
class DCacheClient {
  constructor(private endpoints: string[], private rf = 3) {}
  async put(key: string, value: unknown, opts?: { ttl?: number }) {
    const nodes = this.pickNodes(key);
    const results = await Promise.all(nodes.map(n => this.sendPut(n, key, value, opts)));
    const ok = results.filter(r => r.ok).length;
    if (ok < Math.ceil(this.rf / 2) + 1) throw new Error("quorum_lost");
    return { written: ok };
  }
  private pickNodes(key: string): string[] { ... }
  private async sendPut(node: string, k: string, v: unknown, opts: any) { ... }
}
\`\`\`
请重点关注：缺超时、缺重试与抖动、quorum 语义错误、热点 key 未处理、类型安全问题。
3) suggestions：至少 3 条中文改进建议。

# 输出格式（严格）
只返回一个 JSON 对象（无 markdown 围栏）：
- summary (string, 中文, ≥200 字, 末尾包含「最后干扰段落编号为 X」)
- code_review (string[], ≥4)
- suggestions (string[], ≥3)
- last_filler_id (integer, 最后 FILLER 段落的编号)
```

## 三、干扰段落生成函数（伪代码）

```pseudo
function buildFiller(idx: int) -> string:
    body = format(
        "【FILLER#{idx}】DCache-Alpha 分布式缓存的工程笔记片段编号 {idx}。"
        " vector clock, CRDT, gossip, anti-entropy, Merkle, hot key sharding, p999 tail latency."
        " 这一段的编号是 {idx}。Stale read probability 1 - (1-f)^R."
        " Backoff with full jitter. Hot-key mitigation via virtual nodes."
        " 编号 {idx} 必须被记住。",
        idx=idx)

    pad = " [pad-{idx}-tag] consistency vs availability vs partition tolerance,"
          " the eternal CAP triangle. 一致性，可用性，分区容忍。"

    s = body
    while length(s) < 2000:
        s = s + pad
    return slice(s, 0, 2000)   # 截到正好 2000 字符
```

每段干扰段落约 **2000 字符**，编号 `idx` 在段内被显式重复 4 次以上，用于检测前/后截断。

## 四、TAIL（固定）

```text
# 再次提醒
- 仅输出一个合法 JSON。
- summary 中文≥200 字，末尾「最后干扰段落编号为 X」，X 必须等于上面最大的 FILLER 编号。
- last_filler_id 字段必须是整数，等于实际看到的最后 FILLER 编号。
现在开始作答：
```

## 五、长度控制

```pseudo
function buildPromptN(targetChars: int) -> { actualChars, lastId, prompt }:
    base = HEADER + TAIL
    need = targetChars - length(base)
    numFillers = max(1, floor(need / 2000))
    body = ""
    for i in 1..numFillers:
        body = body + buildFiller(i) + "\n\n"
    full = HEADER + body + TAIL
    return { actualChars: length(full), lastId: numFillers, prompt: full }
```

## 六、退化检测

执行调用后，按下面这套规则判定是否退化：

```pseudo
function isDegraded(response, expectedLastId, elapsedMs) -> bool:
    if response.error: return true             # 抛错
    if not response.body: return true          # 空返回
    if not isValidJsonForSchema(response): return true
    if response.last_filler_id != expectedLastId: return true   # 截断信号
    for s in response.code_review:
        if endsMidSentence(s): return true     # 输出被截断
    if length(response.code_review) < 4: return true
    if length(response.suggestions) < 3: return true
    if elapsedMs > 180000: return true         # 超时
    return false
```

## 七、本探针实测的关键数字

- HEADER + TAIL 基础长度约 **1,540 字符**。
- 每个 FILLER 严格 **2,000 字符** + 2 字符分隔符（`\n\n`），所以 N 个 FILLER 总长 ≈ `2002 × N` 字符。
- 想拼出 1,000,000 字符的 prompt 大约需要 **499 个 FILLER 段落**。
- 想拼出 2,750,000 字符（最后一次成功的输入）大约需要 **1,374 个 FILLER 段落**。
- 中英混合 prompt 在 Claude tokenizer 下实测比值：**≈ 2.798 chars / token**。
