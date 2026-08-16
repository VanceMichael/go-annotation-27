# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

审核通道一限流我们就直接放弃，退避重试完全没发生。

```
$ ./dramactl review submit --series MD-2026-001 --limit-times 2
{
  "calls": 1,
  "exit_code": 1,
  "limit_times": 2,
  "message": "review: 剧目 MD-2026-001 的 machine-scan 第 1 次尝试失败: model: 审核通道限流: 通道每分钟仅允许 2 次调用",
  "ok": false,
  "retries": 0
}
$ echo $?
1
```

`--limit-times 2` 是模拟「前两次调用被限流、之后恢复」的通道。按 README，限流是可重试的，调用方应该沿错误链判定出限流原因、退避后重试，第 3 次就能过。实际 `calls` 是 1、`retries` 是 0，第一次失败就返回了。

还有一个不对的地方：退出码是 1。README 里 1 是「用法错误或未归类的内部错误」，限流重试后仍不成功应该是 7。现在错误信息里明明写着「审核通道限流」，退出码却归到了未归类那一类，HTTP 侧也不是 429 而是 500。

对照现象：

- `--limit-times 0`（不限流）正常通过：`retries` 0、机审加人审两步都成功、退出码 0。
- `--limit-times 1` 同样第 1 次就返回、`retries` 0、退出码 1。

所以只要出现限流，重试和退出码归类两件事一起失灵。

帮我修好，让限流能被正确识别、退避重试并归类到正确的退出码。已有测试跑一遍不要有回归。

## 含 Bug 版本

- 仓库：VanceMichael/go-annotation-27
- 仓库地址：https://github.com/VanceMichael/go-annotation-27.git
- parent SHA：3d38ff9eebe58fc5340b15177a89bca9c7671b1f

## 复现步骤

```bash
git clone -- https://github.com/VanceMichael/go-annotation-27.git bug-repro
cd bug-repro
git checkout --detach 3d38ff9eebe58fc5340b15177a89bca9c7671b1f
go test ./internal/review/ ./internal/cli/ -run "TestRateLimitedIsRetryable|TestSubmitRetriesOnRateLimit|TestChannelErrorExposesUnderlyingCause|TestSubmitRetriesUpToMaxAttempts|TestReviewSubmitRetriesOnThrottle" -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/review/ ./internal/cli/ -run "TestRateLimitedIsRetryable|TestSubmitRetriesOnRateLimit|TestChannelErrorExposesUnderlyingCause|TestSubmitRetriesUpToMaxAttempts|TestReviewSubmitRetriesOnThrottle" -count=1
--- FAIL: TestRateLimitedIsRetryable (0.01s)
    review_test.go:39: 限流错误应判定为可重试, 实际错误 review: 剧目 MD-2026-001 的 machine-scan 第 1 次尝试失败: model: 审核通道限流: 通道每分钟仅允许 1 次调用
--- FAIL: TestChannelErrorExposesUnderlyingCause (0.00s)
    review_test.go:59: 包装后的错误应可判定为 ErrRateLimited, 实际 review: 剧目 MD-T-001 的 machine-scan 第 1 次尝试失败: model: 审核通道限流: 每分钟仅允许 2 次
--- FAIL: TestSubmitRetriesOnRateLimit (0.00s)
    review_test.go:75: 限流 2 次后审核应成功, 实际 review: 剧目 MD-2026-001 的 machine-scan 第 1 次尝试失败: model: 审核通道限流: 通道每分钟仅允许 2 次调用
--- FAIL: TestSubmitRetriesUpToMaxAttempts (0.00s)
    review_test.go:92: 限流 3 次后审核应成功, 实际 review: 剧目 MD-2026-002 的 machine-scan 第 1 次尝试失败: model: 审核通道限流: 通道每分钟仅允许 3 次调用
FAIL
FAIL	microdrama/internal/review	0.056s
--- FAIL: TestReviewSubmitRetriesOnThrottle (0.01s)
    app_test.go:127: 退出码 = 1, 期望 0, stderr=错误: review: 剧目 MD-2026-001 的 machine-scan 第 1 次尝试失败: model: 审核通道限流: 通道每分钟仅允许 2 次调用
FAIL
FAIL	microdrama/internal/cli	0.055s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/review/ ./internal/cli/ -run "TestRateLimitedIsRetryable|TestSubmitRetriesOnRateLimit|TestChannelErrorExposesUnderlyingCause|TestSubmitRetriesUpToMaxAttempts|TestReviewSubmitRetriesOnThrottle" -count=1
--- FAIL: TestRateLimitedIsRetryable (0.00s)
    review_test.go:39: 限流错误应判定为可重试, 实际错误 review: 剧目 MD-2026-001 的 machine-scan 第 1 次尝试失败: model: 审核通道限流: 通道每分钟仅允许 1 次调用
--- FAIL: TestChannelErrorExposesUnderlyingCause (0.00s)
    review_test.go:59: 包装后的错误应可判定为 ErrRateLimited, 实际 review: 剧目 MD-T-001 的 machine-scan 第 1 次尝试失败: model: 审核通道限流: 每分钟仅允许 2 次
--- FAIL: TestSubmitRetriesOnRateLimit (0.00s)
    review_test.go:75: 限流 2 次后审核应成功, 实际 review: 剧目 MD-2026-001 的 machine-scan 第 1 次尝试失败: model: 审核通道限流: 通道每分钟仅允许 2 次调用
--- FAIL: TestSubmitRetriesUpToMaxAttempts (0.00s)
    review_test.go:92: 限流 3 次后审核应成功, 实际 review: 剧目 MD-2026-002 的 machine-scan 第 1 次尝试失败: model: 审核通道限流: 通道每分钟仅允许 3 次调用
FAIL
FAIL	microdrama/internal/review	0.002s
--- FAIL: TestReviewSubmitRetriesOnThrottle (0.00s)
    app_test.go:127: 退出码 = 1, 期望 0, stderr=错误: review: 剧目 MD-2026-001 的 machine-scan 第 1 次尝试失败: model: 审核通道限流: 通道每分钟仅允许 2 次调用
FAIL
FAIL	microdrama/internal/cli	0.003s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

定向测试与全量回归在 linux/amd64、linux/arm64 双架构下均通过。
限流错误可被调用方通过 errors.Is 判定为 model.ErrChannelThrottled，也可通过 errors.As 取到通道错误类型。
限流后按退避重试，通道恢复时最终成功：--limit-times 2 时通道调用 3 次、重试 2 次、退出码 0。
重试次数达到上限仍未成功时返回限流错误，CLI 退出码为 7，HTTP 侧返回 429。
非限流原因（剧目不存在、内容不予播出等）仍不重试，其既有退出码与错误归类保持不变。
