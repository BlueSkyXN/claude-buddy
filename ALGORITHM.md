# Claude Buddy 算法说明

## 概要

本文档记录当前 `index.html` 中实现的 buddy 生成算法。
页面目前支持两种哈希模式：

- `Bun / Wyhash`，默认模式
- `Legacy / FNV-1a`，兼容旧版 HTML

当前默认的 `Bun / Wyhash` 模式，已经和本地 Bun `1.3.11` 做过对比验证；在已测试样本中，与 `Bun.hash()` 结果一致。

## 核心流程

给定一个 `userId`，buddy 的生成流程如下：

1. 拼接 `userId + SALT`
2. 对拼接后的字符串做哈希，得到 32-bit seed
3. 用该 seed 初始化 `mulberry32(seed32)`
4. 按固定顺序消费随机数：
   - `rarity`
   - `species`
   - `eye`
   - `hat`
   - `shiny`
   - `stats`

伪代码如下：

```js
const seed32 = hash(userId + SALT)
const rng = mulberry32(seed32)

const rarity = rollRarity(rng)
const species = pick(rng, SPECIES)
const eye = pick(rng, EYES)
const hat = rarity === 'common' ? 'none' : pick(rng, HATS)
const shiny = rng() < 0.01
const stats = rollStats(rng, rarity)
```

## 哈希算法

### 1. Bun / Wyhash

这是当前默认模式。

- 哈希族：Wyhash 64-bit
- 浏览器中的实现，按 Zig `std.hash.Wyhash` / Bun `Bun.hash()` 行为对齐
- 传给 `mulberry32` 的 seed，是 64-bit 哈希结果的低 32 位

等价思路：

```js
const seed32 = Number(Bun.hash(userId + SALT) & 0xffffffffn)
```

补充说明：

- `index.html` 里包含了浏览器侧的 Wyhash 实现
- 页面启动时会跑 Bun 文档中的自检向量

### 2. Legacy / FNV-1a

这是保留的兼容模式，对应更早的 HTML 版本。

- 哈希族：FNV-1a 32-bit
- 直接作为 `mulberry32` 的 seed 使用

伪代码：

```js
let h = 2166136261
for (let i = 0; i < s.length; i++) {
  h ^= s.charCodeAt(i)
  h = Math.imul(h, 16777619)
}
return h >>> 0
```

## 固定常量

### SALT

```txt
friend-2026-401
```

### Species

总共 18 个：

- `duck`
- `goose`
- `blob`
- `cat`
- `dragon`
- `octopus`
- `owl`
- `penguin`
- `turtle`
- `snail`
- `ghost`
- `axolotl`
- `capybara`
- `cactus`
- `robot`
- `rabbit`
- `mushroom`
- `chonk`

### Rarity

等级列表：

- `common`
- `uncommon`
- `rare`
- `epic`
- `legendary`

权重：

- `common: 60`
- `uncommon: 25`
- `rare: 10`
- `epic: 4`
- `legendary: 1`

对应概率：

- `common`: 60%
- `uncommon`: 25%
- `rare`: 10%
- `epic`: 4%
- `legendary`: 1%

### Eye

总共 6 个：

- `·`
- `✦`
- `×`
- `◉`
- `@`
- `°`

### Hat

总共 8 个：

- `none`
- `crown`
- `tophat`
- `propeller`
- `halo`
- `wizard`
- `beanie`
- `tinyduck`

关键规则：

- 如果 `rarity === 'common'`，则 `hat` 强制为 `none`
- 只有非 `common` 才会从帽子列表中继续随机

这意味着：

- `common` 不会额外消耗一次帽子选择的 RNG
- 所以后续的 `shiny` 和 `stats` 随机序列会随着 rarity 不同而发生偏移

### Shiny

规则：

```js
shiny = rng() < 0.01
```

所以 `shiny` 概率固定为 1%。

### Stats

属性名：

- `DEBUGGING`
- `PATIENCE`
- `CHAOS`
- `WISDOM`
- `SNARK`

rarity floor：

- `common: 5`
- `uncommon: 15`
- `rare: 25`
- `epic: 35`
- `legendary: 50`

生成逻辑：

1. 随机选一个 `peak` 属性
2. 随机选一个 `dump` 属性，且不能和 `peak` 相同
3. 对每个属性：
   - `peak`：高值
   - `dump`：低值
   - 其他属性：围绕 rarity floor 浮动

伪代码：

```js
if (name === peak) {
  value = Math.min(100, floor + 50 + Math.floor(rng() * 30))
} else if (name === dump) {
  value = Math.max(1, floor - 10 + Math.floor(rng() * 15))
} else {
  value = floor + Math.floor(rng() * 40)
}
```

## 页面暴露的搜索参数

当前页面支持这些筛选和运行参数：

- `species`
- `rarity`
- `eye`
- `hat`
- `shiny`
- `mode`
- `hashAlgorithm`
- `bytes`
- `prefix`
- `start`
- `limit`
- `count`

当前默认值：

- `rarity: 'legendary'`
- `shiny: 'any'`
- `mode: 'random'`
- `hashAlgorithm: 'bun'`
- `bytes: 32`
- `prefix: 'user-'`
- `start: 0`
- `limit: 5000000`
- `count: 1`

### Mode

- `random`
  - 随机生成 hex 格式的 `userId`
  - 浏览器优先使用 `crypto.getRandomValues()`
- `sequential`
  - 按 `prefix + (start + i)` 生成 `userId`

## 概率直觉

一些常见组合的大致概率：

- 指定某个 `species`：约 `1 / 18`
- 指定 `legendary`：`1 / 100`
- `legendary + species`：约 `1 / 1800`
- `legendary + species + eye`：约 `1 / 10800`
- `legendary + species + eye + hat`：约 `1 / 86400`
- `legendary + species + eye + hat + shiny`：约 `1 / 8640000`

因此，高约束组合通常需要数百万次尝试才有较高命中概率。

## 验证结论

调试过程中确认过这些事实：

- 旧 HTML 和 Bun 不一致的根因，是旧版 HTML 使用了 FNV-1a，而不是 Bun 哈希
- `mulberry32`、rarity 抽取逻辑、属性生成顺序，本身不是 Bun/Node 差异的根因
- 当前 `Bun / Wyhash` 模式已经和本地 Bun `1.3.11` 对比验证
- 已测试样本中结果为 `0 mismatch`

## 关于早期脚本的结论

之前讨论过两类脚本：

- 一个简化的 Node 脚本，只覆盖 `rarity + species`
- 一个更接近完整逻辑的 Bun 脚本

最终结论：

- 简化 Node 脚本不是完整的 buddy 实现
- 如果目标是和 Bun 行为保持一致，应以 Bun 版本的思路为准
