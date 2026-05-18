# scholarpilot-config

ScholarPilot 客户端**多服务器路由 manifest** 的中心仓库。

## 用途

Tauri 客户端启动时拉这里的 `servers.json` 拿到全量服务器列表，然后并发 ping 测真延迟，选最低的（或用户手动锁定的）作为业务 API base URL。

详细架构见主仓库 `docs/dev-notes/2026-05-18-multi-server-routing-design.md`。

## 一键发布步骤

1. 在 GitHub 建新仓库 `sstar16/scholarpilot-config`（public，只放配置不放代码）
2. 把本目录下 `servers.json` 复制到仓库根
3. 仓库 Settings → Pages → Source: `Deploy from a branch`，Branch: `main` / 根目录
4. 等 1-2 分钟，访问 `https://sstar16.github.io/scholarpilot-config/servers.json` 验证可拉取
5. 客户端启动会自动拉取；新版本 `servers.json` push 后**下次客户端启动**自动生效

## 增删服务器

- 直接改 `servers.json`，commit + push
- 不需要发新客户端版本
- `enabled: false` 可用作灰度（manifest 里挂着但客户端不探测）

## servers.json schema

```json
{
  "version": 1,                          // schema 版本，breaking change 时升
  "updated_at": "2026-05-18T00:00:00Z",  // ISO8601 时间戳，UI 可展示
  "min_client_version": null,            // 预留：未来可强制客户端最低版本
  "servers": [
    {
      "id": "hk-prod-1",                 // 全局唯一，客户端 localStorage 锁定用
      "region": "HK",                    // ISO 区域码 / 自定义短码
      "label": "香港主节点",              // 用户可见名（中文）
      "base_url": "https://api.scholarpilot.top",  // axios baseURL 用
      "ping_path": "/ping",              // 探测 path，默认 /ping（轻量端点）
      "priority": 1,                     // 数字越小越优先（仅冷启动 fallback，EWMA 测出真延迟后失效）
      "enabled": true,                   // false = manifest 挂着但客户端不参与探测
      "tags": ["primary"]                // 自由标签（如 "primary" / "beta" / "planned"）
    }
  ]
}
```

## 字段约定

| 字段 | 类型 | 必填 | 默认 | 说明 |
|---|---|---|---|---|
| `id` | string | ✓ | — | 全局唯一。改 id 等于换新服 |
| `region` | string | ✓ | — | 区域显示用 |
| `label` | string | ✓ | — | 用户可见，中文 |
| `base_url` | string | ✓ | — | HTTPS 完整 URL，无尾斜杠 |
| `ping_path` | string | ✗ | `/ping` | 探测 path，后端必须实现 |
| `priority` | int | ✗ | `0` | 越小越优先（冷启动用） |
| `enabled` | bool | ✗ | `true` | false = 客户端不探测 |
| `tags` | string[] | ✗ | `[]` | 用于未来分组/筛选 |

## 安全

- 客户端只接受 `https://*.scholarpilot.top` 域名（白名单正则校验，防止 manifest 被劫持塞恶意 URL）
- 拉 manifest 走 HTTPS，CF Pages / GitHub Pages 都自带 TLS
- 失败时客户端自动用 localStorage 缓存兜底；缓存也没有时用客户端硬编码的 HK 节点

## 排版规范

每个 server 对象一行一字段，便于 git diff 看变化。改完后跑：

```bash
python -m json.tool servers.json > /tmp/x.json && mv /tmp/x.json servers.json
```

保证格式统一。
