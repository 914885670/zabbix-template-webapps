# Web Apps by HTTP（Zabbix 7.0 模板）

一台主机上跑多个 Web 应用时，用一份清单宏管理探测：每个应用自动生成“可达性 + 页面内容”两个监控项和两条告警，增删应用只改一行宏，不用复制模板。

适用于任何返回 HTTP 的服务（Next.js、Spring Boot、Nginx 静态站、后台管理页都行），**不需要在被监控主机上装 agent**，探测由 Zabbix server/proxy 直接发起。清单条目既可以直连端口（`erp:3000`），也可以指定域名/主机（`erp|erp.example.com:443:/`），后者适合应用只绑 127.0.0.1、由 nginx 反代的场景。

## 监控什么

对清单里的每个应用：

| 监控项 | 判据 | 触发条件 |
|---|---|---|
| `webapp.reach[{#NAME}]` | 状态码落在 `{$WEBAPP.STATUS_CODES}` 内 | `{$WEBAPP.NODATA.PERIOD}` 内拿不到正常响应 |
| `webapp.page[{#NAME}]` | 用正则 `{$WEBAPP.TITLE.REGEXP}` 能从页面里提取到内容（默认取 `<title>`） | 同上，或提取不到内容（端口通但页面没正常渲染） |

两条触发器都是 HIGH，事件名带应用名，并带 `component=webapp`、`app={#NAME}` 两个标签，方便按标签路由到自己的告警动作。

可达性项只请求响应头（`retrieve_mode: HEADERS`，不下载页面，探测更快），存的值经过一层正则只留**最后一行状态行**（如 `HTTP/2 200`、`HTTP/1.1 302`）——跟着 `{$WEBAPP.STATUS_CODES}` 的判定口径一致，重定向链给出的是最终那个码，告警正文里的"当前值"因此一眼能读，不会塞一整坨 header。

## 依赖

- Zabbix **7.0**（6.0 未测试，主要差异在 HTTP agent 字段和 LLD 行为）
- Zabbix server（或负责采集该主机的 proxy）上有 bash
- 被探测地址从 Zabbix server/proxy 直接可达

## 安装

1. 把 `scripts/webapp.discovery` 放到 Zabbix server/proxy 的 externalscripts 目录，加执行权限：

   ```bash
   install -m 755 webapp.discovery /usr/lib/zabbix/externalscripts/webapp.discovery
   ```

   放在 proxy 上时，路径同理（`/usr/lib/zabbix/externalscripts/`）。

2. 导入 `template_app_webapps_by_http.yaml`（Data collection → Templates → Import）。

3. 把模板 `Template App WebApps by HTTP` 链接到目标主机。

4. 在主机上新增宏 `{$WEBAPP.LIST}`，逗号分隔，支持两种条目：

   ```
   # 直连端口：目标主机取 {HOST.IP}
   erp:3000,api:5000:/health,admin:8443:/login

   # 指定域名/主机：适合应用只绑 127.0.0.1、由 nginx 按域名反向代理
   erp|erp.example.com:443:/,api|api.example.com:443:/health
   ```

   第二种写法里协议由 `{$WEBAPP.SCHEME}` 决定；条目不写端口时，https 默认 443、http 默认 80。保存后等一轮发现（默认 10 分钟，临时可把发现规则的间隔改短），项和触发器会自动出现。

## 宏

| 宏 | 默认值 | 说明 |
|---|---|---|
| `{$WEBAPP.LIST}` | 空 | 应用清单。两种格式：`名称:端口[:路径]` 或 `名称|主机:端口[:路径]`；留空则不生成任何项 |
| `{$WEBAPP.SCHEME}` | `http` | `http` 或 `https`；也决定新格式不写端口时的默认端口（https=443、http=80） |
| `{$WEBAPP.PATH}` | `/` | 条目里没写路径时的默认路径 |
| `{$WEBAPP.DELAY}` | `60s` | 每个应用的探测间隔 |
| `{$WEBAPP.TIMEOUT}` | `10s` | 单次请求超时 |
| `{$WEBAPP.STATUS_CODES}` | `200-399` | 视为健康的状态码区间 |
| `{$WEBAPP.NODATA.PERIOD}` | `3m` | 连续多久没数据算故障 |
| `{$WEBAPP.TITLE.REGEXP}` | `<title[^>]*>(.{1,90}?)</title>` | 内容断言正则，取捕获组 1 |

## 发现脚本单独测

```bash
# 参数：清单 / 默认路径 / 默认主机(旧写法用) / 协议(决定缺省端口)
/usr/lib/zabbix/externalscripts/webapp.discovery 'erp:3000,api|api.example.com:443:/health' '/' '10.0.0.10' 'https'
# {"data":[{"{#NAME}":"erp","{#HOST}":"10.0.0.10","{#PORT}":"3000","{#PATH}":"/"},{"{#NAME}":"api","{#HOST}":"api.example.com","{#PORT}":"443","{#PATH}":"/health"}]}
```

## 已知限制（都是实际踩过的）

- **外部脚本的位置**：外部检查类型由 server/proxy 执行，脚本必须在那台机器的 `externalscripts` 目录里，放被监控端无效。
- **key 必须等于脚本文件名**：这里发现规则的 key 是 `webapp.discovery[...]`，所以脚本文件必须叫 `webapp.discovery`，改名后要同步改 key。
- **超时**：Zabbix 的默认超时（3 秒）会截断慢请求。模板已经给项单独设了 `{$WEBAPP.TIMEOUT}`，如果仍然报 timeout，先确认 server 配置里的 `Timeout` 没有被压得太小。
- **目标主机怎么来的**：旧写法 `名称:端口` 的目标固定为 `{HOST.IP}`，这个值由发现规则 key 的第 3 个参数传入（实测 `{HOST.IP}` 在外部检查 key 里会展开）。如果应用只绑回环、由 nginx 按域名反代，用新写法 `名称|域名:端口:路径`，不要把 `{HOST.IP}` 写进用户宏的值。
- **https 默认不校验证书**：模板里 `verify_peer`/`verify_host` 是 0，内部自签环境可以直接用。要校验就打开这两个字段，并确保 CA 在系统库里。
- **内容断言依赖页面里有可提取的内容**：纯 API（返回 JSON）不适合用 `webapp.page`，可以只留可达性项，或者把 `{$WEBAPP.TITLE.REGEXP}` 改成能匹配 JSON 字段的正则。
- **LLD 清理有延迟**：从清单里删掉一个应用后，旧项会在“保留丢失资源”期（3 天）后才消失，这期间它的触发器可能还在。
- 应用名里不能带 `:` 和 `,`，否则清单解析会错位。

## 变更记录

- **v1.1.0（2026-10-26）**：清单新增 `名称|主机:端口[:路径]` 写法，支持应用绑回环、由 nginx 按域名反代的场景；发现脚本新增 `{#HOST}`，LLD key 增加 `{HOST.IP}` 与 `{$WEBAPP.SCHEME}` 两个参数；旧的 `名称:端口[:路径]` 写法保持兼容。
- **v1.0.0（2026-10-25）**：首版。

## 许可证

本仓库分两部分授权：

- 模板文件 `template_app_webapps_by_http.yaml`：**GNU GPL v2.0**，全文见根目录 `LICENSE`。与 Zabbix 官方模板的许可证保持一致。
- `scripts/` 目录下的脚本：**MIT**，全文见 `scripts/LICENSE`。

Copyright (c) 2026 xuuz

## 致谢

最初来自内部运维环境里“一台机器多个 Next.js/Java 应用”的监控需求，验证过 7.0.30。
