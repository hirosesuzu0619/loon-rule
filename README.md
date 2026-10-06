# loon-rule

存放个人使用的 Loon 配置、规则与插件，由 [shadowrocket-rules](https://github.com/hirosesuzu0619/shadowrocket-rules) 改写而来，分流结果与 Shadowrocket 那套保持一致。仓库是公开的，文件里不含订阅链接、节点名、MitM 证书等任何私密内容。

## 与 Shadowrocket 版的结构差异

Shadowrocket 版由一份远程配置加一个本地模块组成，模块里既定义 `US`、`MEXC_TW` 等策略组，也写着指向这些组的规则。Loon 的插件做不到这一点：插件里的规则只能使用 `DIRECT`、`REJECT` 系列和 `PROXY` 三种策略，引用不到配置文件里的策略组。另外，Loon 的配置文件导入后就是设备上的一份本地副本，订阅链接和 MitM 证书也写在里面，不能像 Shadowrocket 的远程配置那样随仓库自动更新。

所以这里把内容拆成三层。`configs/base.conf` 只负责 `[General]` 设置、节点筛选、策略组，以及按顺序引用各个规则文件，导入一次即可，平时很少需要改。所有分流规则都放在 `rules/` 目录，按策略拆成若干 `.list` 文件，由配置的 `[Remote Rule]` 引用，Loon 会定时拉取更新，改规则只需合并到默认分支。`plugins/personal.plugin` 只放 google.cn 跳转这类与策略组无关的 Rewrite 和 MitM 主机名。

Loon 的订阅规则文件只能整体指定一个策略，因此原先 `personal.module` 里混在一起的规则按策略分到了 `Personal_Reject.list`（屏蔽）、`Personal_Direct.list`（直连）、`Personal_US.list`、`Personal_MEXC_TW.list` 和 `Personal_SG.list` 五个文件。拆分后同一服务的规则可能分在不同文件里，例如希尔顿的埋点域 `smetric.hilton.com` 在屏蔽文件里，其余希尔顿域名在美国文件里，各文件的注释里都写明了这种对应关系。

## configs/base.conf

规则的匹配顺序与 Shadowrocket 版一致：个人规则最先，然后是局域网直连、国内域名直连（blackmatrix7 维护的 Loon 版 `China.list` 与 `China_Domain.list`）、常用境外服务走 `EAST_ASIA`、Telegram 走 `TELEGRAM`，以上都没命中时按解析出的 IP 判断，`GEOIP,CN` 直连，其余 `FINAL,EAST_ASIA`。个人规则内部的先后也有讲究：屏蔽排在美国之前，否则 `smetric.hilton.com` 会先被 `hilton.com` 后缀带去 `US`；直连排在美国之前，Apple Music 与 Apple TV+ 共用的 `play.itunes.apple.com`、`bag.itunes.apple.com` 才会先被直连接住。

需要注意 Loon 的两条匹配原则。第一，目标是域名时，Loon 先匹配全部域名规则，都没命中才解析 DNS、再匹配 IP 规则，同类规则之间才按配置顺序比较。第二，规则来源的优先级是本地规则高于插件规则、插件规则高于订阅规则。正因为如此，配置文件的 `[Rule]` 里只留下 `GEOIP,CN` 和 `FINAL` 两条兜底：它们是 IP 规则和最终策略，放在本地不会抢在任何域名规则之前；而如果把常用境外名单写成本地规则，`facebook` 关键字就会压过订阅规则里的 `graph.facebook.com`，Meta AI 的登录接口又会被带去 `EAST_ASIA`。今后新增规则请写进 `rules/` 下对应的文件，不要直接加到配置的 `[Rule]` 里。

节点筛选写在 `[Remote Filter]`，类型是 `NameRegex`，不写订阅名，正则作用于全部节点，包括本地节点和所有订阅，这与 Shadowrocket 版不写 `policy-path` 的效果相同，订阅链接完全不必出现在文件里。正则直接沿用 Shadowrocket 版，同时写了简体、繁体和英文，两字母缩写加 `\b` 单词边界，并用否定前瞻排除名字里带「倍」的倍率节点以及「剩余流量」「套餐到期」「官网」这类信息节点。正则统一用双引号括起来，含逗号也不会被拆开。节点命名风格因机场而异，加载后进分组看一眼实际匹配到的节点，按需补关键字。

策略组在 `[Proxy Group]` 里，每个组引用一个同名加 `_NODES` 后缀的筛选器。`EAST_ASIA` 是通用境外出口，在香港、日本、韩国节点里按延迟自动择优；`TELEGRAM` 在此基础上加入新加坡节点，并改用 `telegram.org` 测速，原因见配置文件注释，简单说就是到 gstatic 的延迟反映不出节点到 Telegram 机房的线路好坏；`US`、`SG`、`JP` 同样是 `url-test`，`tolerance = 50` 让延迟相差不到 50 毫秒时不切换。`MEXC_TW` 是 `select`，筛选器只匹配名为「台湾1」的单个节点，出口完全固定，换节点改 `MEXC_TW_NODE` 那一行的节点名即可；该节点下线时这个组会空掉，走它的流量直接失败而非悄悄换 IP。组名刻意不叫 `PROXY`、`AUTO` 这类常见名字，避免与插件或其他配置来源重名。Shadowrocket 版曾因行尾注释被并进正则导致分组失效，这里同样所有注释都单独成行。

屏蔽 QUIC 的方式也换了。Shadowrocket 版靠排在最前面的 `AND,((PROTOCOL,UDP),(DST-PORT,443)),REJECT-NO-DROP` 规则，但 Loon 会先匹配域名规则，这条逻辑规则排不到 x.com 等域名规则前面，而且 Loon 没有 `REJECT-NO-DROP` 策略，所以改用 `[General]` 里的 `disable-udp-ports = 443`，作用于全部流量，不受规则顺序影响。`udp-fallback-mode = REJECT` 对应 Shadowrocket 的 `udp-policy-not-supported-behaviour = REJECT`，节点不支持 UDP 时拒绝而不是改成直连。DNS 使用腾讯与阿里的 DoH，失败时 Loon 回落到系统 DNS 和普通 DNS；`ip-mode = ipv4-only` 关闭 IPv6，避免请求绕开节点从本机 IPv6 地址直接出去。

## rules/

`Overseas.list` 是常用境外服务名单，内容是 Google、YouTube、GitHub、Reddit、Netflix 等常见站点的关键字和后缀，按域名直接命中，每个新域名省去一轮 DNS 查询，被污染的解析结果也不影响分流。没有引用 blackmatrix7 的 `Global` 或 `Proxy` 名单，因为它们包含 `apple.com`、`icloud.com`、`microsoft.com`、`akamai.net` 等域名，会把 iCloud、系统更新和国内也在用的 CDN 一并改走代理。`Telegram.list` 是 Telegram 的域名与官方 `cidr.txt` 公布的 IP 段。

五个 `Personal_*.list` 的内容和 Shadowrocket 版 `personal.module` 一一对应，共 114 条，覆盖 Siri 与 Apple 隐私中继、Apple TV+、Apple Music、X、Meta AI、Claude、OpenAI、Gemini、Grok、MEXC、Bybit、希尔顿、Kraken、Kalshi 等，每条规则的来由都保留在文件注释里。写法是 `类型,值`，不带策略，策略由配置里引用该文件的那一行决定；IP 规则末尾加 `no-resolve`。

## plugins/personal.plugin

把 `google.cn`、`g.cn`（含 `www.` 前缀）用 302 跳转到 `https://www.google.com`，跳转由 Loon 在本地直接返回，不经过任何节点。正则在主机名之后要求紧跟 `/`、`:`、`?` 或网址结尾，否则 `g.cn.miaozhen.com` 这类以 `g.cn` 开头的其他域名也会被误跳转。对 https 地址，不解密就看不到完整 URL，所以插件同时把这四个主机名加进 MitM 列表；证书仍用设备上那份配置里生成的。没有安装并信任证书时，只有 http 地址的跳转会生效。Rewrite 用的是 Loon 3.5.1 之前的旧语法，新旧版本都能识别。

## 使用方法

在 Loon 里通过 URL 导入配置，地址填 `https://raw.githubusercontent.com/hirosesuzu0619/loon-rule/HEAD/configs/base.conf`。导入后在 App 里添加机场订阅，再到 MitM 页面生成并信任证书，这些内容只保存在设备上的配置里。规则和插件已经写在配置里，会随仓库自动更新，无需另外添加。

导入之后先进各个策略组确认节点匹配正常，尤其是 `MEXC_TW` 里是否恰好有一个节点。若设备上另有其他插件或订阅规则，检查它们有没有引用同名策略组，或者用本地规则抢先接走了这里的域名：本地规则的优先级高于订阅规则，配置 `[Rule]` 里额外加的域名规则会压过 `rules/` 下的所有文件。在本地另存的副本请命名为 `*.local.conf`，已被 `.gitignore` 忽略。
