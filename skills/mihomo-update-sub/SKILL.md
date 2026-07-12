---
name: mihomo-update-sub
description: 更新 mihomo 机场订阅配置。当用户提到"更新订阅"、"更新机场"、"更新代理节点"、"更新mihomo订阅"时使用此 skill。
version: 1.0.0
---

# Mihomo 机场订阅更新

从机场订阅链接拉取最新代理节点配置，替换 mihomo 的 config.yaml。

## 前置条件

- mihomo 已安装并运行
- 用户提供有效的机场订阅链接（通常为 `https://xxx.com/api/v1/client/subscribe?token=xxx` 格式）
- Python3 可用（用于解析订阅内容）

## 执行步骤

### 第 1 步：确认订阅链接

向用户确认订阅链接。当前系统使用的订阅链接格式示例：
```
https://login.yfjc.xyz/api/v1/client/subscribe?token=xxx
```

如果用户未提供，询问订阅链接。

### 第 2 步：下载订阅内容

```bash
# 将 <订阅链接> 替换为实际链接
curl -sL "<订阅链接>" -o /tmp/subscription.txt

# 检查下载结果
wc -c /tmp/subscription.txt
```

正常内容应为 base64 编码的文本，文件大小通常在 10-50KB。

### 第 3 步：解码并检查格式

```bash
# 解码订阅内容
base64 -d /tmp/subscription.txt > /tmp/subscription_decoded.txt

# 查看前几行，确认格式
head -5 /tmp/subscription_decoded.txt
```

常见的协议格式：
- `hysteria2://password@server:port/?params#name`
- `vless://uuid@server:port?params#name`
- `ss://...`（Shadowsocks）
- `trojan://...`（Trojan）

### 第 4 步：转换订阅为 mihomo YAML

使用 Python 脚本解析订阅 URI 并生成 mihomo 配置：

```bash
cat << 'PYEOF' > /tmp/gen_config.py
import base64
import urllib.parse
import yaml
import sys

with open('/tmp/subscription.txt', 'r') as f:
    content = f.read().strip()

try:
    decoded = base64.b64decode(content).decode('utf-8')
except:
    decoded = content

lines = [l.strip() for l in decoded.split('\n') if l.strip()]
print(f"共解析到 {len(lines)} 个节点", file=sys.stderr)

proxies = []
proxy_names = []

for line in lines:
    try:
        if line.startswith('hysteria2://'):
            rest = line[len('hysteria2://'):]
            password_part, rest = rest.split('@', 1)
            password = urllib.parse.unquote(password_part)
            
            if '?' in rest:
                server_port, params = rest.split('?', 1)
                if '#' in params:
                    params, frag = params.split('#', 1)
                    frag = urllib.parse.unquote(frag)
                else:
                    frag = ''
            else:
                server_port = rest
                params = ''
                frag = ''
            
            server_port = server_port.rstrip('/')
            server, port = server_port.rsplit(':', 1)
            port = int(port)
            
            qs = urllib.parse.parse_qs(params)
            sni = qs.get('sni', [''])[0]
            insecure = qs.get('insecure', ['false'])[0] in ('true', '1')
            pin_sha256 = qs.get('pinSHA256', [''])[0]
            mport = qs.get('mport', [''])[0]
            
            proxy = {
                'name': frag if frag else f'{server}:{port}',
                'type': 'hysteria2',
                'server': server,
                'port': port,
                'password': password,
                'sni': sni,
                'skip-cert-verify': insecure,
            }
            if pin_sha256:
                proxy['pin-sh256'] = pin_sha256
            if mport:
                proxy['mport'] = mport
            
            proxies.append(proxy)
            proxy_names.append(proxy['name'])
        
        elif line.startswith('vless://'):
            rest = line[len('vless://'):]
            uuid_part, rest = rest.split('@', 1)
            uuid = urllib.parse.unquote(uuid_part)
            
            if '?' in rest:
                server_port, params = rest.split('?', 1)
                frag = ''
                if '#' in params:
                    params, frag = params.split('#', 1)
                    frag = urllib.parse.unquote(frag)
            else:
                server_port = rest
                params = ''
                frag = ''
            
            server_port = server_port.rstrip('/')
            server, port = server_port.rsplit(':', 1)
            port = int(port)
            
            qs = urllib.parse.parse_qs(params)
            sni = qs.get('sni', [''])[0]
            insecure = qs.get('insecure', ['false'])[0] in ('true', '1')
            pin_sha256 = qs.get('pinSHA256', [''])[0]
            flow = qs.get('flow', ['xtls-rprx-vision'])[0]
            network = qs.get('type', [''])[0] or qs.get('network', [''])[0]
            ws_path = qs.get('path', [''])[0]
            ws_host = qs.get('host', [''])[0] or qs.get('sni', [''])[0]
            
            proxy = {
                'name': frag if frag else f'{server}:{port}',
                'type': 'vless',
                'server': server,
                'port': port,
                'uuid': uuid,
                'tls': True,
                'servername': sni,
                'flow': flow,
                'skip-cert-verify': insecure,
            }
            if pin_sha256:
                proxy['pin-sh256'] = pin_sha256
            if network == 'ws':
                proxy['network'] = 'ws'
                if ws_path:
                    proxy['ws-opts'] = {
                        'path': ws_path,
                        'headers': {'Host': ws_host} if ws_host else {}
                    }
            
            proxies.append(proxy)
            proxy_names.append(proxy['name'])
    except Exception as e:
        print(f"跳过: {line[:80]}... 错误: {e}", file=sys.stderr)

# 读取现有配置以保留自定义设置
with open('/home/wq/.config/mihomo/config.yaml', 'r') as f:
    old_config = yaml.safe_load(f)

# 保留原有的 DNS、规则、代理组等配置
proxy_groups = old_config.get('proxy-groups', [])
rules = old_config.get('rules', [])

# 重建代理组，使用新节点
new_groups = []
for group in proxy_groups:
    if group.get('name') == '一分机场':
        group['proxies'] = proxy_names
        new_groups.append(group)
    elif group.get('name') in ['日本自动选择', '日本故障转移']:
        group['proxies'] = proxy_names
        new_groups.append(group)
    else:
        # 保留其他代理组但更新节点列表
        group['proxies'] = proxy_names
        new_groups.append(group)

# 去重代理组名称
seen = set()
unique_groups = []
for group in new_groups:
    name = group.get('name')
    if name not in seen:
        seen.add(name)
        unique_groups.append(group)

# 更新规则中的代理组引用
for i, rule in enumerate(rules):
    rules[i] = rule.replace('一分机场', '日本专线')

# 构建新配置
config = {
    'allow-lan': old_config.get('allow-lan', False),
    'bind-address': old_config.get('bind-address', '*'),
    'dns': old_config.get('dns', {}),
    'geoip-dat-url': old_config.get('geoip-dat-url', 'https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geoip.dat'),
    'geosite-dat-url': old_config.get('geosite-dat-url', 'https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geosite.dat'),
    'ipv6': old_config.get('ipv6', True),
    'log-level': old_config.get('log-level', 'info'),
    'mixed-port': old_config.get('mixed-port', 7897),
    'mode': old_config.get('mode', 'rule'),
    'profile': old_config.get('profile', {'store-selected': True}),
    'proxies': proxies,
    'proxy-groups': unique_groups,
    'rules': rules,
    'tcp-concurrent': old_config.get('tcp-concurrent', True),
    'tun': old_config.get('tun', {
        'auto-detect-interface': True,
        'auto-route': True,
        'dns-hijack': ['any:53'],
        'enable': True,
        'stack': 'system',
        'strict-route': False,
    }),
    'unified-delay': old_config.get('unified-delay', True),
}

with open('/home/wq/.config/mihomo/config.yaml', 'w') as f:
    yaml.dump(config, f, allow_unicode=True, default_flow_style=False, sort_keys=False)

print(f"✅ 成功生成 {len(proxies)} 个节点", file=sys.stderr)
PYEOF

python3 /tmp/gen_config.py
```

检查生成的配置：
```bash
grep "name:" /home/wq/.config/mihomo/config.yaml | head -20
```

### 第 5 步：备份当前配置

```bash
cp /home/wq/.config/mihomo/config.yaml /home/wq/.config/mihomo/config.yaml.bak
```

### 第 6 步：验证配置合法性

```bash
/home/wq/.local/bin/mihomo -d /home/wq/.config/mihomo/ -f /home/wq/.config/mihomo/config.yaml -t 2>&1 | grep -E "test is successful|Parse config error"
```

如果显示 `test is successful`，继续下一步。否则停止并报告错误。

### 第 7 步：重启 mihomo

```bash
kill $(pgrep mihomo) 2>/dev/null
sleep 3
nohup /home/wq/.local/bin/mihomo -d /home/wq/.config/mihomo/ -f /home/wq/.config/mihomo/config.yaml > /tmp/mihomo_sub_update.log 2>&1 &
sleep 5
```

### 第 8 步：验证服务

```bash
# 检查端口
ss -tlnp | grep 7897

# 测试代理
curl -s -o /dev/null -w "代理测试: HTTP %{http_code} | %{time_total}s\n" https://www.google.com --proxy http://127.0.0.1:7897 --connect-timeout 10
```

### 第 9 步：回滚机制

如果服务启动失败或代理测试失败，立即回滚：

```bash
# 恢复备份
cp /home/wq/.config/mihomo/config.yaml.bak /home/wq/.config/mihomo/config.yaml

# 重启服务
kill $(pgrep mihomo) 2>/dev/null
sleep 2
nohup /home/wq/.local/bin/mihomo -d /home/wq/.config/mihomo/ -f /home/wq/.config/mihomo/config.yaml > /dev/null 2>&1 &
sleep 5

# 验证
ss -tlnp | grep 7897
curl -s -o /dev/null -w "回滚后代理测试: HTTP %{http_code} | %{time_total}s\n" https://www.google.com --proxy http://127.0.0.1:7897 --connect-timeout 10
```

### 第 10 步：清理

验证成功后，删除临时文件和备份：

```bash
rm -f /home/wq/.config/mihomo/config.yaml.bak
rm -f /tmp/subscription.txt /tmp/subscription_decoded.txt /tmp/gen_config.py /tmp/mihomo_sub_update.log
```

## 注意事项

1. **订阅链接安全**：订阅链接包含敏感 token，不要在日志或文档中暴露完整链接
2. **节点去重**：转换脚本会自动去重代理组名称
3. **保留自定义配置**：脚本会保留原有的 DNS、规则、TUN 等配置
4. **协议兼容性**：
   - Hysteria2 节点：需要确保 `skip-cert-verify` 设置正确
   - VLESS Reality 节点：可能需要特殊的 TLS 配置
5. **备份重要性**：更新前必须备份，以便回滚

## 输出格式

更新完成后，向用户报告：
- 订阅链接（部分脱敏）
- 解析到的节点总数
- 保留的自定义配置（DNS、规则等）
- 服务状态
- 代理测试结果

## 常见问题处理

### 订阅解析失败

如果订阅内容无法解析：
1. 检查订阅链接是否有效
2. 确认订阅内容格式（base64 编码）
3. 查看错误日志

### 节点全部失效

如果更新后所有节点都无法连接：
1. 检查 `skip-cert-verify` 设置
2. 确认节点协议类型（Hysteria2/VLESS）
3. 联系机场客服确认节点状态

### 配置冲突

如果出现配置冲突：
1. 保留原有配置的 DNS、规则部分
2. 只替换 proxies 和 proxy-groups
3. 手动合并冲突的设置项
