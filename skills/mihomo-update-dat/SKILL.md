---
name: mihomo-update-dat
description: 更新 mihomo 地址库（geoip.dat 和 geosite.dat）。当用户提到"更新地址库"、"更新 mihomo 数据库"、"更新 geoip"、"更新 geosite"时使用此 skill。
version: 1.0.0
---

# Mihomo 地址库更新

更新 mihomo 的 GeoIP 和 GeoSite 数据库文件到最新版本。

## 前置条件

- mihomo 已安装并配置完成
- 配置文件路径：`/home/wq/.config/mihomo/config.yaml`
- 数据库文件路径：`/home/wq/.config/mihomo/geoip.dat`、`/home/wq/.config/mihomo/geosite.dat`

## 执行步骤

### 第 1 步：检查当前版本

```bash
echo "=== 本地数据库文件信息 ==="
ls -lh /home/wq/.config/mihomo/geoip.dat /home/wq/.config/mihomo/geosite.dat
```

获取远程最新版本信息：
```bash
curl -sL "https://api.github.com/repos/MetaCubeX/meta-rules-dat/releases/latest" | python3 -c "
import json, sys
data = json.load(sys.stdin)
print(f\"发布版本: {data.get('tag_name', '未知')}\")
print(f\"发布时间: {data.get('published_at', '未知')}\")
for asset in data.get('assets', []):
    if 'geoip' in asset['name'] or 'geosite' in asset['name']:
        print(f\"  {asset['name']}: {asset['size']/1024/1024:.1f} MB\")
"
```

对比本地和远程版本：
- 比较文件大小
- 比较最后修改日期

### 第 2 步：备份当前数据库

```bash
cp /home/wq/.config/mihomo/geoip.dat /home/wq/.config/mihomo/geoip.dat.bak
cp /home/wq/.config/mihomo/geosite.dat /home/wq/.config/mihomo/geosite.dat.bak
```

### 第 3 步：下载最新数据库

```bash
cd /tmp
curl -sL "https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geoip.dat" -o geoip.dat
curl -sL "https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geosite.dat" -o geosite.dat
```

校验文件大小（正常大小：geoip.dat 约 17-19MB，geosite.dat 约 4MB）：
```bash
GEOIP_SIZE=$(stat -c%s /tmp/geoip.dat)
GEOSITE_SIZE=$(stat -c%s /tmp/geosite.dat)
if [ "$GEOIP_SIZE" -lt 10000000 ] || [ "$GEOSITE_SIZE" -lt 1000000 ]; then
    echo "❌ 下载失败：文件太小"
    exit 1
fi
```

### 第 4 步：替换数据库文件

```bash
mv /tmp/geoip.dat /home/wq/.config/mihomo/geoip.dat
mv /tmp/geosite.dat /home/wq/.config/mihomo/geosite.dat
```

### 第 5 步：重启 mihomo 服务

```bash
kill $(pgrep mihomo) 2>/dev/null
sleep 3
nohup /home/wq/.local/bin/mihomo -d /home/wq/.config/mihomo/ -f /home/wq/.config/mihomo/config.yaml > /dev/null 2>&1 &
sleep 5
```

### 第 6 步：验证服务

```bash
ss -tlnp | grep 7897
curl -s -o /dev/null -w "HTTP: %{http_code} | %{time_total}s\n" https://www.google.com --proxy http://127.0.0.1:7897 --connect-timeout 10
```

### 第 7 步：回滚机制

如果服务启动失败或代理测试失败，立即回滚：
```bash
cp /home/wq/.config/mihomo/geoip.dat.bak /home/wq/.config/mihomo/geoip.dat
cp /home/wq/.config/mihomo/geosite.dat.bak /home/wq/.config/mihomo/geosite.dat
kill $(pgrep mihomo) 2>/dev/null
sleep 2
nohup /home/wq/.local/bin/mihomo -d /home/wq/.config/mihomo/ -f /home/wq/.config/mihomo/config.yaml > /dev/null 2>&1 &
sleep 5
```

### 第 8 步：清理

验证成功后，删除备份文件：
```bash
rm -f /home/wq/.config/mihomo/geoip.dat.bak /home/wq/.config/mihomo/geosite.dat.bak
```

## 注意事项

1. 下载的是 `latest` 版本的数据库，会覆盖本地文件
2. 更新后必须重启 mihomo 才能生效
3. 如果下载失败（网络问题），会自动使用旧的备份文件
4. 数据库文件较大（约 20MB），下载可能需要几秒钟
5. 建议定期更新（每月一次），以保持最新的 IP 和域名规则

## 输出格式

更新完成后，向用户报告：
- 本地旧版本文件大小和修改日期
- 远程最新版本文件大小和发布日期
- 是否成功更新
- 代理测试结果
