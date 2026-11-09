# adblock-rules

把两个广告规则源自动合并转换成 **Quantumult X** 可订阅的分流 / 重写规则文件，并额外产出 **AdGuard / AdGuard Home** 可订阅的黑名单与白名单：

1. [VME98/jinx-rules](https://github.com/VME98/jinx-rules)——Jinx iOS DNS 过滤 App 的规则仓库（域名黑白名单 + URL 广告规则）。
2. [uxudjs/Shadowrocket 的 fuck_apps_ad_sr.sgmodule](https://raw.githubusercontent.com/uxudjs/Shadowrocket/refs/heads/main/modules/fuck_apps_ad_sr.sgmodule)——各大 App 去广告模块中的 [Rule] 与 [URL Rewrite] 段。

两个源转换时统一去重合并，所以**不会重复拦截同样的域名**。

- GitHub Actions 每 **6 小时**拉取上游规则并转换一次；产物头部会写「最近更新」时间和本次新增条数，规则没变时新增为 0。
- 转换器采用**安全模式**：宁可少拦、不可误杀。无法安全转换的规则写入 `generated/unsupported.log` 并在 Actions 日志中提示数量。
- **单文件交付**：QX 分流只需订阅一个文件；AdGuard 侧输出黑名单 + 白名单两个文件。

## 订阅地址

```
https://raw.githubusercontent.com/wahaha1024/adblock-rules/main/jinx-adblock-qx.list
```

在 Quantumult X → 分流 → 引用（`[filter_remote]`）中添加即可。文件内白名单（direct）在前、黑名单（reject）在后，**不要**对该远程资源设置 `force-policy=reject`——那会让白名单失效。

中国大陆网络直连 raw.githubusercontent.com 可能不通，可改用 jsDelivr CDN：

```
https://cdn.jsdelivr.net/gh/wahaha1024/adblock-rules@main/jinx-adblock-qx.list
```

## 可选：URL 重写文件

上游还有约 750 条完整 URL 广告规则（如 `https://xxx/api/getAdverList`），这类规则无法放进分流文件（QX 分流只认域名/IP），已转成 QX 重写规则单独发布：

```
https://raw.githubusercontent.com/wahaha1024/adblock-rules/main/generated/jinx-qx-rewrite.conf
```

在 QX → 重写 → 引用（`[rewrite_remote]`）中添加。HTTPS 重写需要开启 MITM；参考 [`generated/jinx-mitm-skip.txt`](generated/jinx-mitm-skip.txt) 把不应解密的域名（`*.apple.com` 等）从你的 `[mitm] hostname` 中排除。MITM 配置始终手工维护，本仓库不会自动改写你的 `[mitm]` 段。

## 可选：AdGuard / AdGuard Home 订阅

同一份黑白名单另出 **AdGuard DNS 过滤语法**（adblock 风格）的两个文件，AdGuard Home、AdGuard DNS 等都能直接订阅：

```
# 黑名单（拦截）
https://raw.githubusercontent.com/wahaha1024/adblock-rules/main/generated/adguard-blocklist.txt

# 白名单（放行，全部是 @@ 例外规则）
https://raw.githubusercontent.com/wahaha1024/adblock-rules/main/generated/adguard-allowlist.txt
```

jsDelivr 镜像（大陆网络）：

```
https://cdn.jsdelivr.net/gh/wahaha1024/adblock-rules@main/generated/adguard-blocklist.txt
https://cdn.jsdelivr.net/gh/wahaha1024/adblock-rules@main/generated/adguard-allowlist.txt
```

在 AdGuard Home 里这样加：

1. **Filters → DNS blocklists → Add blocklist → Add custom list**，粘贴黑名单 URL；
2. **Filters → DNS allowlists → Add allowlist → Add custom list**，粘贴白名单 URL。
   旧版没有 allowlists 分区时，把白名单也加进 blocklists 即可——文件里全是 `@@` 例外规则，一样按放行处理；也可以把内容整段粘进 Custom filtering rules。

语法映射（与 QX 文件同一批桶，去重/白名单排除逻辑完全一致）：

```
host, a.com, reject                 ->  |a.com^         只拦 a.com 本身
host-suffix, a.com, reject          ->  ||a.com^        拦 a.com 及其所有子域
host-wildcard, p*-ad.a.com, reject  ->  ||p*-ad.a.com^  * 匹配任意字符
host-keyword, umeng, reject         ->  ||*umeng*^      域名任意位置包含该串
ip-cidr, 1.2.3.4/32, reject         ->  （不输出）       DNS 过滤没有 IP 维度
任意 direct 规则                     ->  加 @@ 前缀      例外规则，优先级最高
```

注意：

- **精确域名只拦本身**：`|a.com^` 不含子域，和 QX 的 `host` 语义一致；子域拦截由上游 `*.a.com` 规则负责。想让某条精确规则连子域一起拦，把开头的 `|` 改成 `||`。
- **白名单优先级最高**：AdGuard 的例外规则压过所有黑名单（包括你订阅的其他规则），与 QX 文件里 direct 排在最前等价。
- **16 条 `ip-cidr` 规则不会出现在 AdGuard 文件里**：DNS 过滤只认域名，没有 IP 维度；这些规则仍在 QX 文件中生效。
- **`sdk.e.qq.com` 等 9 个广告 SDK 域名**属于 QX 精准处理层（连接层 direct、广告端点靠 URL 重写拦截）。AdGuard 没有 URL 重写能力，如果你希望 DNS 层直接拦死它们，把 allowlist 里对应的 `@@` 行删掉。
- 23 条 keyword 规则转成 `||*keyword*^`，匹配范围与 QX `host-keyword` 相同，粒度比精确域名粗。

### 第三方规则列表的远程转换

AdGuard Home 不认识 QX / Clash 语法。把 `host-suffix, x, reject` 这种列表直接订阅进去，界面会显示"已加载 N 条规则"，实际一条也匹配不到——行数照数，语法一条没懂。本仓库在 CI 里把常用的第三方列表转换成 AdGuard 语法，订阅转换后的产物即可：

| 产物 | 上游 | 规模 |
|---|---|---|
| `generated/adguard-fmz200.txt` | [fmz200/wool_scripts](https://github.com/fmz200/wool_scripts) `QuantumultX/filter/filter.list`（QX 语法，黑名单） | 约 2600 条 |
| `generated/adguard-unbreak.txt` | [ddgksf2013/Filter](https://github.com/ddgksf2013/Filter) `Unbreak.list`（Clash 语法，白名单） | 约 30 条 |
| `generated/adguard-anudeep-allow.txt` | [anudeepND/whitelist](https://github.com/anudeepND/whitelist) `domains/whitelist.txt`（纯域名，白名单） | 约 190 条 |

订阅方式与上面两个文件相同：黑名单进 DNS blocklists，白名单进 DNS allowlists。三个文件头部都写了上游 URL、源语法和被跳过的条数，可以直接搬去别的 DNS 过滤客户端，不依赖本仓库的其它文件。

- IP-CIDR、USER-AGENT、URL 正则这类 DNS 层无法表达的规则会被跳过，数量写在头部 `# lines skipped`。
- `hl2guide/AdGuard-Home-Whitelist` 本身就是 AdGuard 语法，无需转换，直接订阅 `https://cdn.jsdelivr.net/gh/hl2guide/AdGuard-Home-Whitelist@main/whitelist.txt`。
- 想加新源：在 `config/sources.json` 的 `dns_sources` 里追加一项（`name` / `url` / `syntax`：`quantumultx` | `clash` | `domains` / `mode`：`block` | `allow` / `output`），转换器会抓取、转换、校验并写进 `generated/`。
- 抓取顺序是 `url` → `mirror`（默认配了 jsDelivr），两者都失败只告警并保留旧文件，不会连累自家产物；解析出的条数低于 `min_rules` 也按失败处理——这是防 raw.githubusercontent 返回截断内容把列表悄悄变短。

## 文件清单

| 文件 | 用途 |
|---|---|
| `jinx-adblock-qx.list` | **主订阅文件**：合并后的域名分流（白名单 direct → 黑名单 reject） |
| `generated/adguard-blocklist.txt` | 可选：**AdGuard 黑名单**（adblock 语法，订阅到 DNS blocklists） |
| `generated/adguard-allowlist.txt` | 可选：**AdGuard 白名单**（`@@` 例外规则，订阅到 DNS allowlists） |
| `generated/adguard-*.txt` | 可选：第三方列表（fmz200 / Unbreak / anudeepND）的 AdGuard 转换产物，见上节 |
| `generated/jinx-qx-rewrite.conf` | 可选：URL 正则重写（含 jinx URL 规则 + sgmodule URL Rewrite 段） |
| `generated/jinx-mitm-required.txt` | 可选：**重写生效所需解密的域名**（当前 527 条，随上游重写规则增减，含 `.oneline.txt` 单行粘贴版） |
| `generated/jinx-mitm-skip.txt` | 可选：MITM 排除域名参考 |
| `generated/qx-config-snippet.conf` | 可选：可直接粘进 QX 配置的三段配置片段 |
| `generated/unsupported.log` | 转换审计：被跳过的规则及原因 |

生成文件头部带上游 commit、最近更新时间与本次新增条数（首次生成显示总条数），可直接溯源，也能一眼看出这次订阅到底变没变。**转换器自带自校验**：产物格式错误、规则重复、白名单顺序颠倒、正则编译失败都会让 CI 直接失败，不会把坏文件推给你。

> 完整的架构与设计说明见 [docs/DESIGN.md](docs/DESIGN.md)。

> sgmodule 的 `[Script]`（101 条远程脚本）、`[Body Rewrite]`（45 条 jq 改写）是 Surge/Shadowrocket 专属能力，QX 没有等价物，因此不会出现在生成文件中——已在 `unsupported.log` 列出。想要这部分效果请继续在 Surge/Shadowrocket 里启用该模块，两边并不冲突。

## 与其他规则叠加的顺序建议

QX 分流自上而下匹配。若同时使用 AdRules 等广告规则与本文件：

```ini
[filter_remote]
; 1. 本文件（自带 direct 放行，须先于其他广告规则）
; 2. 其他去广告规则（如有）
; 3. 规则修正 / 去广告修正规则（放最后）
```

原则：**修正规则（direct 放行）永远排在最后**，才能覆盖前面规则的拦截。

> GitHub 域名默认**不在**本文件的放行名单里（`output_exclude` 已剔除），这样 GitHub 走你自己的代理分流；若你希望 GitHub 直连，自行加一条 `host-suffix, github.com, direct` 到 `[filter_local]` 即可。

## 与 AdRules 的 jinx 模块对比

[adrules.bittersweetx.dpdns.org 的 shared_log_jinx.sgmodule](https://adrules.bittersweetx.dpdns.org/modules/shared_log_jinx.sgmodule) 是同一个 jinx 上游的 Surge 手工精修版。用 `python tools/compare_adrules.py` 实测（当前上游数据）：

| 维度 | AdRules 模块 | 本仓库 |
|---|---|---|
| 域名拦截 | 3686（exact + wildcard） | 3882，多 168 条 |
| URL 重写 | 374 | 853，其中 315 条与 AdRules 重叠 |
| 需解密域名 | 360 条（手工维护） | 527 条（`jinx-mitm-required.txt` 自动生成） |
| 转换不了的部分 | — | QX 无 `AND+NOT` 原子规则能力 |

AdRules 独有的三样东西，其中两样已补进本仓库：

- **`wpad`、`sdkquic.e.qq.com`**：前者的裸名单标签、后者的 GDT QUIC 端点 jinx 上游没有——已加入 `config/manual_extras.list` 手工补充。
- **端口容错**：AdRules 的重写模式都带 `(?::[0-9]+)?` 以匹配显式端口，本仓库的重写规则现已同样带上该可选端口组（此前只匹配默认端口）。
- **GDT/Pangle SDK 精准处理**（已移植）：AdRules 让 9 个广告 SDK 域名（`sdk.e.qq.com`、`mi.gdt.qq.com`、`api-access.pangolin-sdk-toutiao*.com` 等）连接层 DIRECT，仅用重写拦截其广告端点；并整体拒绝 `sdkquic.e.qq.com`、`webcast-open.douyin.com` 两个 QUIC/UDP 重度域名。这些已进入 `config/manual_extras.list` 精准处理层：分流文件里 9 条 `host, ..., direct`（在拦截规则之前生效），重写文件里 3 条 GDT/Pangle 广告端点拦截（`get_ads`、`pre_fetch`、`gdt_mview.fcg`，HTTPS 生效需 MITM）。
- **`AND,((NETWORK,UDP),(DEST-PORT,443),(DOMAIN,mi.gdt.qq.com)),REJECT`** 这类按"域名+协议+端口"的 UDP 精准拦截：依赖 Surge 的 `extended-matching` 原子规则，**QX 分流没有对应语法**。QX 侧的做法见下一节。

AdRules 的 `AND+NOT` 白名单豁免（`DOMAIN-WILDCARD,adx.*.com` 减掉 `*.duolingo.com` 等）同样是 Surge 专属能力。本仓库在**转换阶段**用白名单把对应黑名单规则直接排除，效果等价且不占运行时规则。

## 已配置 MITM 还能优化什么

QX 的 MITM 已解密 HTTPS 时，下面这些开关能让广告拦截更彻底：

1. **把 `generated/jinx-mitm-required.txt` 加进 `[mitm] hostname`**（527 条）。不解密的域名，重写文件里针对 HTTPS 的规则一条都不会触发——这是「MITM 配了但广告还在」的头号原因。两种用法：
   - 逐行复制 `jinx-mitm-required.txt`；
   - 或直接抄 `jinx-mitm-required.oneline.txt`（已逗号连接的单行，可整段粘贴）。
   想缩短清单，把后缀填进 `config/sources.json` 的 `mitm_fold_suffixes`（例如 `"qishui.com"`），转换器会把该后缀下 ≥3 个域名折叠成一条 `*.qishui.com`。默认不折叠——折叠会放宽解密范围，由你决定。
2. **丢弃 QUIC，逼 App 走可拦截的 TCP**：HTTP/3 走 UDP 443 加密，MITM 证书对它无效，是广告规则漏网的主要原因。QX 官方配置项（写进 `[general]`）：

   ```ini
   udp_drop_list = QUIC
   ```

   只想拦 443 的 QUIC 就配 `udp_whitelist = 53, 80, 123, 443` 再加上面的 `udp_drop_list`。副作用是部分 App 首次连接慢半拍（QUIC 超时后回落 TCP），YouTube 等强依赖 QUIC 的服务可能受影响。
 3. **GDT/Pangle 的 QUIC 端点**：AdRules 用 `AND,((NETWORK,UDP),(DEST-PORT,443),(DOMAIN,mi.gdt.qq.com)),REJECT` 精准掐断，QX 没有 `AND` 语法。本仓库已移植其精准处理层（`config/manual_extras.list`）：9 个 SDK 域名连接层 direct、广告端点走重写拦截，`sdkquic.e.qq.com` / `webcast-open.douyin.com` 整体拒绝；再配合第 2 条的全局 QUIC 丢弃即可等效。
4. **排除法维护 `[mitm]`**：`jinx-mitm-skip.txt` 里的 33 个域名（支付、系统验证、iCloud 等）务必排除，否则会出现验证码加载失败、支付异常。
5. **叠加修正规则放最后**：与 AdRules 或其他去广告规则共用时，把 direct 放行类修正规则排在最后，才能覆盖前面的拦截。

## 转换规则

上游 → QX 映射（详见 `converter/convert.py`）：

| 上游写法 | QX 规则 |
|---|---|
| `ads.example.com` | `host, ads.example.com, reject` |
| `*.example.com` | `host-suffix, example.com, reject` |
| `p*-ad.adkwai.com`（中间通配） | `host-wildcard, p*-ad.adkwai.com, reject` |
| `DOMAIN-KEYWORD,umeng,REJECT`（sgmodule） | `host-keyword, umeng, reject` |
| `IP-CIDR,1.2.3.4/32,REJECT`（sgmodule） | `ip-cidr, 1.2.3.4/32, reject` |
| `^https?://... - reject-dict`（sgmodule） | `^https?://... url reject-dict` |
| URL 白名单 | 转换阶段直接从黑名单中排除 |
| 通用路径（`/api/pay/` 等，无 host） | **跳过**（safe_mode，误杀面太大） |
| 泛路径（`/splash`、`/api/ad` 等） | **跳过**（safe_mode） |
| 空路径 URL | 降级为 host 规则 |
| 正则形 URL | **跳过**（`convert_regex: false`） |

白名单永远排在黑名单前面输出——QX 规则顺序敏感，先匹配先生效。

### 宽泛通配的误杀收窄

上游和 Shadowrocket 模块里各有一条通配，命中面覆盖了正经域名而不只是广告域名：

| 规则 | 来源 | Tranco top-1M 命中 | 典型误杀 |
|---|---|---|---|
| `ad*.*.*` | `fuck_apps_ad_sr.sgmodule` | 407 | adidas.co.uk、adobe.co.jp、adelaide.edu.au、adm.gov.it、adjust.net.in |
| `*.ad.*` | 上游 `blacklist_wildcard.txt` | 79 | 全部是 `.ad.jp`（日本 JPNIC 二级域：rakuten.ad.jp、iij.ad.jp、nifty.ad.jp…） |

两条都在 `config/sources.json` 的 `wildcard_drop` 里剔除，改由 `config/manual_extras.list` 里 11 条收窄前缀接管（`ads.*.*`、`adserver.*.*`、`adservice.*.*`、`adtech.*.*`、`adclick.*.*`、`adtrack.*.*`、`adzone.*.*`、`adx.*.*`、`adv.*.*`、`adnxs.*.*`、`adform.*.*`；top-1M 只命中 26 个域名，全是广告/追踪域）。剔除只针对这两条，其余 164 条通配（`ad-*`、`*-ads.*`、`ads-*`、`*-ad.*` 等）保持原样——它们的精度尚可，`ad-*` 一类仍会命中少量正经站点（如 ad-hoc-news.de），需要更严格时可以照同样方式加进 `wildcard_drop`。

### 两个源的重复情况

实测重叠极少（用 `python tools/compare_sources.py` 可直接复现）：

- 域名规则：sgmodule 的 6 条 `DOMAIN-SUFFIX` 里只有 `mmstat.com` 与 jinx 完全重复，4 条 `DOMAIN-WILDCARD` 里只有 `ad.*` 重复，1 条 `DOMAIN` 无重复；
- 16 条 `IP-CIDR`（滴滴/贴吧的服务端 IP）是 jinx 完全没有的，属净增拦截；
- 23 条 `DOMAIN-KEYWORD`（umeng、adservice 等）粒度比 jinx 粗，但 jinx 侧往往只有少量精确域名命中同一关键词——合并后覆盖面互补，实测 762 条重写规则重复全部来自 jinx 内部三份合并版/拆分版文件；
- URL 重写：107 条中与 jinx 的 890 条 URL 规则仅 2 条近似。

两个源粒度不同例：`doubleclick.net` 在 sgmodule 是整域后缀拦截，在 jinx 是 `ad.doubleclick.net` 等精确条目，合并后按更粗的那条生效。

## 本地开发

```bash
# 从上游下载并转换
python converter/convert.py

# 用本地 checkout 转换（调试用）
python converter/convert.py --source-dir /path/to/jinx-rules

# 测试
python -m pip install pytest
python -m pytest tests/ -q
```

配置在 `config/sources.json`：上游仓库、分支、源文件清单与安全开关（`safe_mode` / `convert_generic_path` / `convert_unknown_url` / `convert_regex` / `deduplicate`）。

## 致谢与声明

- 规则内容全部来自 [VME98/jinx-rules](https://github.com/VME98/jinx-rules)，本仓库只做格式转换。
- 转换器在安全模式下只输出"确定安全"的规则；被跳过的规则都记录在 `unsupported.log` 中，可自行评估后补充。
