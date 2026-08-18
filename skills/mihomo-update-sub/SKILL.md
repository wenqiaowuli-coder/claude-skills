---
name: mihomo-update-sub
description: 更新 mihomo 机场订阅（订阅+主配置同步），含备份、验证、回滚机制。当用户提到"更新机场订阅"、"更新mihomo"、"更新代理配置"、"更新订阅"、"更新机场"、"更新代理节点"、"更新mihomo订阅"时触发。
version: 2.0.0
---

# Mihomo 机场订阅完整更新

两步走流程：更新机场订阅 → 同步到主配置（仅日本线路），全程带备份/验证/回滚。

## 前置条件

- mihomo 已安装并以 systemd 用户服务运行（`mihomo.service`）
- 订阅脚本：`~/.config/mihomo/scripts/update-yfjc-sub.sh`
- 配置同步脚本：`~/.config/mihomo/scripts/sync-config.py`
- 主配置：`~/.config/mihomo/config.yaml`
- 代理提供者：`~/.config/mihomo/proxy-providers/yfjc.yaml`
- Python3 + PyYAML 可用

## 架构说明

```
~/.config/mihomo/
├── config.yaml              # 主配置（DNS、规则、代理组、TUN）
├── proxy-providers/
│   └── yfjc.yaml            # 机场订阅节点（由脚本更新）
├── scripts/
│   ├── update-yfjc-sub.sh   # 订阅更新脚本
│   └── sync-config.py       # 配置同步脚本
└── backup/                  # 自动备份目录
```

## 执行步骤

### 第 1 步：更新订阅

```bash
bash ~/.config/mihomo/scripts/update-yfjc-sub.sh
```

脚本自动完成：下载订阅 → Base64 解码 → 转 YAML → 备份 → 写入 `proxy-providers/yfjc.yaml` → 重启 mihomo → 验证代理。

回滚：若 mihomo 重启失败，自动恢复最新备份的 `yfjc.yaml`。

### 第 2 步：同步到主配置

```bash
python3 ~/.config/mihomo/scripts/sync-config.py
```

脚本自动完成：
1. **备份**：`config.yaml` → `backup/config.yaml.bak.<timestamp>`
2. **清理**：移除 proxies 段中的无效条目（防御性清理）
3. **读取节点**：从 `proxy-providers/yfjc.yaml` 读取所有节点
4. **过滤**：排除流量/到期/重置等非节点信息条目 + 黑名单节点
5. **筛选**：**仅保留 🇯🇵 日本节点**，丢弃其他所有区域
6. **修复字段**：`skip-cert-verify: true`（所有节点，兼容 legacy CN 证书）、`client-fingerprint` 转换、`tls: true`（Reality 节点）
7. **重建 proxies**：写入有效节点
8. **重建 proxy-groups**：
   - `日本-Hysteria2` 组：url-test，所有日本 hysteria2 节点（5个）
   - `VLESS-Reality` 组：url-test，所有日本 VLESS Reality 节点（9个）
   - `VLESS-CF` 组：url-test，所有日本 VLESS CF-Warp 节点（6个）
   - `日本专线` 组：fallback，`日本-Hysteria2` → `VLESS-Reality` → `VLESS-CF`（故障自动降级）
9. **修正规则**：自动将引用不存在组的规则改为 `日本专线`
10. **保留**：DNS、tun 等原有配置
11. **验证**：`mihomo -t` 检查配置语法
12. **重启**：`systemctl --user restart mihomo`
13. **连通性**：测试 YouTube/Google 可访问性

回滚：任何步骤失败 → 恢复 `backup/config.yaml.bak.<timestamp>` → 重启 mihomo。

## 订阅链接

当前使用的订阅链接（硬编码在 `update-yfjc-sub.sh` 中）：

```
https://login.yfjc.xyz/api/v1/client/subscribe?token=<YOUR_TOKEN>
```

如需更换订阅链接，直接修改脚本中的 `SUB_URL` 变量。

## 输出格式

更新完成后，向用户报告：
- 订阅链接（部分脱敏）
- 解析到的节点总数
- 节点协议分布（Hysteria2 / VLESS）
- 备份文件路径
- 服务状态
- 代理测试结果

## 节点筛选规则

同步脚本通过 `KEEP_REGIONS` 和 `NODE_BLACKLIST` 控制。

### KEEP_REGIONS
仅保留指定区域的节点，当前：
```python
KEEP_REGIONS = {"日本专线"}
```

| 筛选条件 | 处理方式 |
|----------|----------|
| 🇯🇵 日本节点 | ✅ 保留 |
| 🇺🇸🇸🇬🇹🇼🇬🇧🇳🇱🇫🇷🇩🇪🇧🇷🇰🇷 等其他区域 | ❌ 丢弃 |
| 剩余流量/套餐到期/距离下次重置 | ❌ 丢弃（非节点信息） |

### NODE_BLACKLIST
排除已知不稳定或高延迟的节点，当前：
```python
NODE_BLACKLIST = {"🇯🇵日本专线5-0.1倍率"}
```

如需恢复黑名单中的节点，从 `NODE_BLACKLIST` 中删除后重新同步即可。

## 代理组路由架构

### 组结构
```
规则 (google.com/youtube.com/github.com ...)
  → 日本专线 (fallback)
    1. 日本-Hysteria2 (url-test: 5 个 hysteria2 节点，可测真实延迟，默认优先)
    2. VLESS-Reality (url-test: 9 个 Reality 节点，延迟显示 0ms，Hysteria2 故障时降级)
    3. VLESS-CF (url-test: 6 个 CF-Warp 节点，前两者都故障时降级)
    → 全部失败 → DIRECT (直连)
```

### 设计原则
- **协议分层**：按协议类型拆分节点组（Hysteria2 / VLESS-Reality / VLESS-CF），避免不同特性的节点混在同一组内测速。
- **组引用优于个体节点**：`日本专线` 引用 3 个子组名，而非直接列出所有 20 个节点。避免 url-test 对不可测延迟的 Reality 节点反复超时。
- **fallback 容灾**：`日本专线` 使用 fallback 模式，优先使用 Hysteria2，故障时自动降级到 Reality，再故障降级到 CF-Warp。

## 节点兼容性修复

同步脚本自动处理以下兼容性问题：

| 问题 | 修复方式 |
|------|----------|
| Go 1.26 拒绝 legacy CN 证书 | 所有节点 `skip-cert-verify: true` |
| mihomo v1.19+ `fingerprint` 字段变更 | 自动转为 `client-fingerprint` |
| VLESS Reality 节点缺少 `tls` | 自动添加 `tls: true` |
| proxies 段混入代理组名等无效条目 | 自动清理（需 `type` + `server` 字段） |

## systemd 服务优化

`mihomo.service` 已配置端口释放延迟，避免重启时端口冲突：

```ini
[Service]
Type=simple
ExecStart=/home/wq/.local/bin/mihomo -d /home/wq/.config/mihomo/ -f /home/wq/.config/mihomo/config.yaml
Restart=on-failure
RestartSec=5
TimeoutStopSec=5
ExecStopPost=/bin/sleep 1
```

修改后需执行 `systemctl --user daemon-reload` 生效。

## 验证结果

完成后检查：

```bash
# 服务状态
systemctl --user status mihomo

# 连通性
export http_proxy=http://127.0.0.1:7890
curl -s -o /dev/null -w "%{http_code}\n" https://www.youtube.com
```

## 日志文件

| 文件 | 用途 |
|------|------|
| `~/.config/mihomo/logs/yfjc-update.log` | 订阅更新日志 |
| `~/.config/mihomo/logs/sync-config.log` | 配置同步日志 |

## 备份文件

- `~/.config/mihomo/backup/yfjc.yaml.bak.<timestamp>` — 订阅备份
- `~/.config/mihomo/backup/config.yaml.bak.<timestamp>` — 主配置备份

## 当前保留线路

同步完成后，config.yaml 中包含：

| 代理组 | 模式 | 节点数 | 说明 |
|--------|------|--------|------|
| 日本-Hysteria2 | url-test | 5 | hysteria2 专线节点（-0.1倍率） |
| VLESS-Reality | url-test | 9 | VLESS Reality 节点（移动专属/通用） |
| VLESS-CF | url-test | 6 | VLESS CF-Warp 节点（日本1-6号） |
| 日本专线 | fallback | 3 | → Hysteria2 → VLESS-Reality → VLESS-CF |

## 常见问题

**订阅解析失败**：
1. 检查订阅链接是否有效（可浏览器访问）
2. 查看脚本日志：`~/.config/mihomo/logs/yfjc-update.log`
3. 手动下载检查：`curl -sL <订阅链接> | base64 -d | head -5`

**节点全部失效**：
1. 检查 `skip-cert-verify` 设置
2. 确认节点协议类型和参数
3. 联系机场客服确认节点状态

**mihomo 启动失败**：
1. 检查配置语法：`mihomo -t`
2. 查看 systemd 日志：`journalctl --user -u mihomo -n 30`
3. 常见错误：
   - `invalid REALITY short id`：short-id 格式错误，应为纯十六进制
   - `duplicate name`：节点名称重复，需确保唯一
   - `bind: address already in use`：端口被占用

**配置冲突**：如果手动修改了 `config.yaml`，更新订阅时注意：
- `proxy-providers/yfjc.yaml` 不受影响
- 手动同步节点到 `config.yaml` 时会覆盖原有节点列表
- DNS、规则、TUN 等配置会保留

**YAML 中 emoji 显示为 `\U0001F1EF`**：正常现象，YAML 的 Unicode 转义，mihomo 能正确解析。

**需要恢复其他区域节点**：修改 `sync-config.py` 中的 `KEEP_REGIONS` 后重新同步。

**新增日本节点不显示**：先更新订阅，再运行同步脚本即可。

**hysteria2 服务器不可达**：订阅更新后服务器地址可能变更。如果所有 hysteria2 节点都不可达，检查 `日本-Hysteria2` 组是否全部超时。如果全部失效，可能需要等待机场恢复或手动切换节点。

**VLESS Reality 节点不可用**：Reality 节点的 url-test 延迟显示为 0ms，这是正常的（mihomo 无法对 Reality 协议做延迟测试）。如果 Reality 节点全部失败，通常是服务器端证书或网络问题，需等待机场修复。

**VLESS-CF 节点不可用**：CF-Warp 节点走 Cloudflare 边缘，延迟通常较高。如果该组全部失败，可能是 Cloudflare 节点被墙，可暂时从 `VLESS-CF` 组中移除该节点。

**故障自动降级**：`日本专线` 使用 fallback 模式，默认优先 Hysteria2。如果 Hysteria2 故障，自动降级到 VLESS-Reality；如果 Reality 也故障，再降级到 VLESS-CF。

## 相关文件

| 文件 | 说明 |
|------|------|
| `~/.config/mihomo/scripts/update-yfjc-sub.sh` | 订阅更新脚本 |
| `~/.config/mihomo/scripts/sync-config.py` | 配置同步脚本 |
| `~/.config/mihomo/config.yaml` | 主配置文件 |
| `~/.config/mihomo/proxy-providers/yfjc.yaml` | 机场订阅节点 |
| `~/.config/mihomo/backup/` | 备份目录 |
| `~/.config/systemd/user/mihomo.service` | systemd 服务配置（含端口释放延迟） |
