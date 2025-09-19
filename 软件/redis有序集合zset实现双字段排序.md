# redis有序集合zset实现爽字段排序

相关代码: github.com/chaunsin/example/internal/pkg/ranking

## redis有序集合score分数限制

1. 分数大小限制

Redis 排序集使用双精度 64 位浮点数来表示分数。在我们支持的所有架构中，分数都表示为IEEE 754 浮点数，能够精确表示介于-(2^53)
和+(2^53)之间的整数。更实际地说，-9007199254740992 和 9007199254740992
之间的所有整数都可以完美表示。较大的整数或分数在内部以指数形式表示，因此您得到的分数可能只是小数或非常大整数的近似值。

https://redis.io/docs/latest/commands/zadd/#range-of-integer-scores-that-can-be-expressed-precisely

因此分数值要在-9007199254740992 和 9007199254740992之间，不然会存在丢失精度问题。

2. 分值排序限制

当排序分值一样时，次是redis列表排序规则是按照字典形式进行排序。也就是根据member成员值。

https://redis.io/docs/latest/commands/zadd/#elements-with-the-same-score

## field1降序 + field2升序

## field1降序 + field2降序

## 利用redis自身字典顺序特性实现排序。

```go
package redisrank

import (
	"fmt"
	"math"
	"strconv"
	"strings"
)

const (
	// 100 年大约的秒数：100 * 365.25 * 24 * 3600 ≈ 3_155_760_000
	windowSeconds = 3155760000

	// base 时间戳（Unix 秒），可选业务起点；只要保证所有 ts-baseTS ∈ [0, windowSeconds)
	baseTS = int64(0)

	// 用于格式化 inverted 的宽度（windowSeconds-1 最大 10 位左右）
	invertedWidth = len(strconv.FormatInt(windowSeconds-1, 10))
)

// EncodeEntry 生成 ZADD 用的 score 和 member。
//  - primary: 主字段（如收益值），≥0，最好 ≤ 2^53-1.
//  - ts: Unix 时间戳（秒），保证 ts-baseTS ∈ [0, windowSeconds).
//  - id: 唯一标识，如用户ID、记录ID等，不能包含 ':'。
// 返回：
//   score  = float64(primary)
//   member = fmt.Sprintf("%0*d:%s", invertedWidth, inverted, id)
func EncodeEntry(primary, ts int64, id string) (score float64, member string, err error) {
	rel := ts - baseTS
	if rel < 0 || rel >= windowSeconds {
		return 0, "", fmt.Errorf("timestamp %d out of 100y window", ts)
	}
	inverted := (windowSeconds - 1) - rel
	// 拼成固定宽度十进制 + ':' + id
	member = fmt.Sprintf("%0*d:%s", invertedWidth, inverted, id)
	return float64(primary), member, nil
}

// DecodeEntry 从 member 里解析出 primary（从 score 传入）和 ts、id。
//  - score: ZSCORE 拿回的 float64，Round 后即 primary。
//  - member: 上面格式化出的字符串。
// 返回：primary, ts (Unix秒), id, error
func DecodeEntry(score float64, member string) (primary, ts int64, id string, err error) {
	primary = int64(math.Round(score))

	// 拆 inverted:id
	parts := strings.SplitN(member, ":", 2)
	if len(parts) != 2 {
		err = fmt.Errorf("invalid member format")
		return
	}
	inverted, err := strconv.ParseInt(parts[0], 10, 64)
	if err != nil {
		return
	}
	rel := (windowSeconds - 1) - inverted
	ts = baseTS + rel
	id = parts[1]
	return
}
```

**注意：member得数值要放到开头，从而满足字典顺序。**

当插入数据时

``` go
// 构建分值和member成员
var(
    value=9876543210
    timestamp=1721059200
    userId=user123
)
score, member, err := redisrank.EncodeEntry(value, timestamp, userId)
if err != nil {
    panic(err)
}

// 插入到redis中
```

查询topN

``` go
// 查询 TopN：
res, err := client.ZRevRangeWithScores("rank:zset", 0, N-1).Result()
for _, z := range res {
    // 对数据进行解析还原
    primary, ts, id, _ := redisrank.DecodeEntry(z.Score, z.Member.(string))
    fmt.Println("收益", primary, "达成于", time.Unix(ts,0), "用户", id)
}
```

更新分值，当需要更新分值时，不能使用`ZINCRBY`命令增加分值,因为需要同时更新member得key才行，才能满足排序规则。
因此实际操作中需要执行如下代码

- 找到并删除旧的 composite‑member
- 插入新的 composite‑member（带最新 timestamp 的那条

具体代码如下：

```go
// 使用 go-redis/v8 举例
ctx := context.Background()
zkey := "rank:zset"
oldMember := "0001234567:user123"
newMember := "0001234560:user123"
newScore  := 9876543210.0

pipe := rdb.TxPipeline()
pipe.ZRem(ctx, zkey, oldMember)
pipe.ZAdd(ctx, zkey, &redis.Z{Score: newScore, Member: newMember})
_, err := pipe.Exec(ctx)
if err != nil {
// 处理错误
}
```

或lua脚本

``` go
// -- KEYS[1]: zset key
// -- ARGV[1]: oldMember
// -- ARGV[2]: newScore
// -- ARGV[3]: newMember
var updateRank = redis.NewScript(`
  redis.call("ZREM", KEYS[1], ARGV[1])
  redis.call("ZADD", KEYS[1], ARGV[2], ARGV[3])
  return 1
`)

func UpdateEntry(ctx context.Context, rdb *redis.Client,
    zkey, oldMember, newMember string, newScore float64,
) error {
    // KEYS[1] = zkey
    // ARGV[1] = oldMember
    // ARGV[2] = strconv.FormatFloat(newScore, 'f', -1, 64)
    // ARGV[3] = newMember
    _, err := updateRank.Run(ctx, rdb, []string{zkey},
        oldMember,
        fmt.Sprintf("%f", newScore),
        newMember,
    ).Result()
    return err
}
```

建议使用lua脚本，一次io同时也能保证原子性。

以下是带原子操作版本

```lua
-- KEYS[1] = zset key
-- ARGV[1] = delta        -- 增减量，可正可负
-- ARGV[2] = newTs        -- 新的 Unix 秒
-- ARGV[3] = id           -- 记录 ID
-- ARGV[4] = oldMember    -- 旧的 composite member
-- 脚本中用常量 windowSeconds & baseTS，也可以传进来
local zkey = KEYS[1]
local delta = tonumber(ARGV[1])
local newTs = tonumber(ARGV[2])
local id    = ARGV[3]
local oldM  = ARGV[4]

-- 1. 原子地增值并获取旧分值
local oldScore = redis.call("ZSCORE", zkey, oldM)
if not oldScore then
  return redis.error_reply("member not exist")
end
local newScore = oldScore + delta

-- 2. 生成新的 composite member
local rel = newTs - baseTS
if rel < 0 or rel >= windowSeconds then
  return redis.error_reply("timestamp out of window")
end
local inverted = (windowSeconds - 1) - rel
-- 假设 invertedWidth 已知
local fmt = "%0" .. invertedWidth .. "d:" .. "%s"
local newM = string.format(fmt, inverted, id)

-- 3. 删除旧 member、添加新 member
redis.call("ZREM", zkey, oldM)
redis.call("ZADD", zkey, newScore, newM)

return { tostring(newScore), newM }
```

## 相关redis命令使用

# 参考

- https://blog.csdn.net/z69183787/article/details/113310091