# Egern 规则集

从 [blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script) 自动构建的 Egern 格式规则集。

## 输出结构

每个分类会产出两个文件，数据同源，避免漂移：

```
Rules/
└── Apple/
    ├── Apple.yaml        # 完整版，给 rules: 里的 rule_set 用
    └── Apple_DNS.yaml    # DNS 精简版，只保留 domain 相关字段，
                          # 给 dns.forward 里的 proxy_rule_set 用
```

`Apple_DNS.yaml` 只保留 `domain_set` / `domain_suffix_set` / `domain_keyword_set` /
`domain_wildcard_set`，剔除了 `ip_cidr_set`、`geoip_set`、`asn_set`、`user_agent_set`、
`dest_port_set` 等 DNS 阶段用不上的字段（这些是解析完成后才有意义的信息，
或者是 HTTP 层信息，DNS 只处理域名本身）。

两个文件在构建时共用同一份原始数据，DNS 版本是从完整版 YAML 直接提取生成的，
不是独立下载/独立维护，因此不会出现两者不同步的问题。

某个分类如果完全不含 domain 类规则（纯 IP/ASN 分类），则不会生成对应的 `_DNS.yaml`。

## 引用方式

```yaml
# rules 部分
rules:
  - rule_set:
      match: https://raw.githubusercontent.com/wot1026-cmd/egern/main/Rules/Apple/Apple.yaml
    policy: DIRECT

# dns 部分
dns:
  forward:
    - proxy_rule_set:
        match: https://raw.githubusercontent.com/wot1026-cmd/egern/main/Rules/Apple/Apple_DNS.yaml
        value: bootstrap
```

## 相比 Repcz/EgernRules 原版的改动

以下改动均经过逐条实测验证，不是凭空猜测：

| 改动 | 原因 |
|---|---|
| sparse-checkout 只拉 `rule/Surge` | 减少下载体积，不需要 Clash/Loon/QuantumultX 等其他目录 |
| IP-CIDR/IP-CIDR6 去重时剥离 no-resolve 后比较 | 实测确认同一网段带/不带 no-resolve 会被误判成两条不同规则保留，造成冗余 |
| Clone 失败发 Telegram 告警 | 原脚本失败后无任何通知，问题发现滞后 |
| 新增 DNS 精简版双文件输出 | 供 `dns.forward` 精细化分流使用 |

以下"疑似问题"经实测确认**不成立**，原逻辑保留未改动：

- URL-REGEX 正则替换：实测对带引号/不带引号两种输入都能正确处理
- awk 规则类型前缀匹配顺序：逗号分隔符让前缀天然唯一，不存在误判可能
- `no_resolve` 全局开关设计：查证 Egern 官方文档，该字段本就是**文件级**、仅对 IP 类规则（geoip/ip_cidr/ip_cidr6/asn）生效，不影响 domain 部分，原逻辑正确

## Telegram 告警配置（可选）

如果要启用 Clone 失败告警，在仓库 Settings → Secrets and variables → Actions 里添加：

- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_CHAT_ID`

不配置的话，构建失败时只会在 Actions 日志里体现，不影响正常构建流程。

## 手动触发构建

Actions 标签页 → Build → Run workflow
