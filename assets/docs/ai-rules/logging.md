# Logging Rules

## 基本规则

- 后端必须使用结构化日志，不使用零散的 `console.log` 作为长期日志方案。
- 日志必须帮助定位问题：包含事件名、请求 ID、用户 ID 或租户 ID、关键资源 ID、耗时和结果。
- 日志不得记录密码、token、cookie、密钥、验证码、完整银行卡号、完整身份证号等敏感信息。
- 错误日志记录内部细节，API 响应返回安全文案。

## 日志级别

- `debug`：本地调试和临时诊断，生产默认可关闭。
- `info`：关键业务事件和状态变化。
- `warn`：可恢复异常、降级、重试、异常输入。
- `error`：请求失败、任务失败、不可恢复异常。

## 结构化日志示例

```ts
// apps/api/src/lib/logger.ts
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  redact: [
    "req.headers.authorization",
    "req.headers.cookie",
    "password",
    "passwordHash",
    "token",
    "accessToken",
    "refreshToken",
    "secret"
  ]
});
```

## 请求日志

每个请求应有稳定的 `requestId`。如果上游传入 `x-request-id`，优先沿用；否则生成新的。

```ts
// apps/api/src/middleware/request-context.ts
import { randomUUID } from "node:crypto";

export function getRequestId(req: Request) {
  return req.headers.get("x-request-id") ?? randomUUID();
}
```

```ts
// apps/api/src/routes/users.ts
import { logger } from "../lib/logger";

export async function createUserHandler(req: Request): Promise<Response> {
  const requestId = getRequestId(req);
  const startedAt = Date.now();

  try {
    const result = await createUser(await req.json());

    logger.info({
      event: "user.create.success",
      requestId,
      userId: result.id,
      durationMs: Date.now() - startedAt
    });

    return Response.json({ ok: true, data: result });
  } catch (error) {
    logger.error({
      event: "user.create.failure",
      requestId,
      durationMs: Date.now() - startedAt,
      error
    });

    throw error;
  }
}
```

## 审计日志

涉及权限、资金、账号安全、数据删除、管理员操作时，必须记录审计事件。

```ts
logger.info({
  event: "audit.user.role_changed",
  actorUserId,
  targetUserId,
  fromRole,
  toRole,
  requestId
});
```

## 禁止项

- 禁止记录完整请求 body，除非已经明确脱敏。
- 禁止记录认证头、cookie、session、token。
- 禁止只输出字符串日志，例如 `logger.info("created user")`。
- 禁止用日志代替错误处理或测试。
