# mihomo-proxy

把「多机场订阅 + 手动选机场 + 自动择优节点」的 [mihomo](https://github.com/MetaCubeX/mihomo)（Clash.Meta 内核）做成**源码一张清单、一条命令部署**的项目。

```
本仓库（源码，你改这里）  ──deploy──▶  /etc/mihomo（部署产物，服务读这里）
```

## 一条铁律

**改配置只改本仓库，永远不要手改 `/etc/mihomo`。**

`/etc/mihomo` 里除了 mihomo 自己写的 `cache.db`，其它内容都能由本仓库重新生成；手改会在下次 `deploy` 被覆盖。

---

## 1. 目录结构

```
mihomo-proxy/
├── mihomo-proxy                 # 【唯一入口】deploy / restart / status / remove
├── README.md                    # 本文件
├── SKILL.md                     # 给 AI agent 的标准部署流程
├── subscriptions.list           # 【订阅清单】日常改这个
├── pkg/
│   └── mihomo-linux-amd64-v3-v1.19.33.deb   # 离线内核包
└── config/                      # ← 整个目录会同步到 /etc/mihomo
    ├── config.yaml              # 运行配置（端口/DNS/TUN/规则），含详细注释
    ├── rules/
    │   └── custom_direct.yaml   # 【自定义直连清单】加直连域名改这个
    ├── proxies/proxy1..5.yaml   # 订阅缓存（缺失时补种，不覆盖更新的）
    ├── geosite.dat              # 域名分类库（GEOSITE 用）
    ├── Country.mmdb             # IP→国家库（GEOIP 用）
    └── ui/                      # 网页控制台静态文件
```

| 你可能会改的文件 | 用途 |
|---|---|
| `subscriptions.list` | 加/删/换机场订阅，以及每个机场用自动选还是手动选 |
| `config/config.yaml` | 端口、DNS、TUN、分流规则、地理库更新开关 |
| `config/rules/custom_direct.yaml` | 强制直连的域名清单 |

---

## 2. 四条命令

```bash
./mihomo-proxy deploy      # 部署 / 更新 / 修复（幂等，需要 root 时自动提权）
./mihomo-proxy restart     # 先检查，全部通过才强制重启（改 TUN / 端口后用）
./mihomo-proxy status      # 只读查询，不改任何东西
./mihomo-proxy rollback    # 回滚到最近一份快照（可指定：rollback /etc/mihomo.bak.2026...）
./mihomo-proxy remove      # 卸载（打印删除清单，要输入 yes 确认）
```

### deploy 做了什么

| 步 | 内容 | 说明 |
|---|---|---|
| 1 | 环境检查 | 架构、AVX2、源码文件完整性（`config.yaml`/规则集/地理库/ui 都得在） |
| 2 | 内核 | 已装且版本等于 `v1.19.33` → 跳过；否则用 `pkg/*.deb` 安装，没有本地包才去 GitHub 下载 |
| 3 | 渲染 + 校验 | `subscriptions.list` → `config.yaml` 的 AUTOGEN 区，然后 `mihomo -t`。**校验不过就退出，线上一个字节都不动** |
| 4 | 比较差异 | 源码 vs `/etc/mihomo` 逐文件比对（`cache.db` 不参与） |
| 5 | 快照 + 同步 | 整目录快照 `/etc/mihomo.bak.<时间戳>`（保留最近 3 份），`cache.db` 与 `.deploy-mode` 带过来；按整棵树补齐差异文件（`ui/`、地理库不会丢）；**同步完先在部署目录里再 `mihomo -t` 一次，失败自动回滚快照** |
| 6 | DNS | 判定 `udcp` / `plain`，写 drop-in 或 `/etc/resolv.conf`；已正确则不改 |
| 7 | 生效 | 无改动 → 只自检；改了配置 → `PUT /configs` 热重载（不断流）；内核刚装 / `listen` 变 / 服务没在跑 → 重启；热重载失败自动退回重启 |
| 8 | 自检 | 服务、内核版本、API、Web UI、TUN 网卡、DNS 解析、7890 代理、自定义规则集，共 8 项 |

**回滚**：

```bash
./mihomo-proxy rollback                            # 回滚到最近一份快照
./mihomo-proxy rollback /etc/mihomo.bak.2026...    # 指定某一份（status 会列出最近的一份）
```

回滚的保护：先校验快照里的配置（`mihomo -t`）不通过就什么都不做；换之前把**当前部署**存成一份新快照，所以回滚本身也可以再回滚一次。

### remove 删什么

打印清单并要求输入 `yes`，然后：

1. `systemctl disable --now mihomo`
2. `dpkg --purge mihomo`（内核 + systemd 单元）
3. 删除 `/etc/mihomo` 与所有 `/etc/mihomo.bak.*`
4. 回滚 systemd-resolved 的 drop-in、恢复 `/etc/resolv.conf`
5. 自检 DNS 是否恢复正常

**不动**：本仓库目录、桌面代理设置。若之前手动配过系统代理，自己恢复：`gsettings set org.gnome.system.proxy mode 'none'`。

---

### 无终端环境（脚本 / CI）怎么给 sudo 密码

`deploy` / `restart` / `rollback` / `remove` 在需要 root 时会自己调 `sudo`；如果不在终端里跑，设置 `SUDO_ASKPASS` 即可（脚本会自动给 `sudo` 加 `-A`）：

```bash
cat > /tmp/askpass.sh <<'EOF'
#!/bin/sh
sed -n 's/^machine sudo login [^ ]* password //p' "$HOME/.netrc"
EOF
chmod 700 /tmp/askpass.sh
SUDO_ASKPASS=/tmp/askpass.sh ./mihomo-proxy deploy
```

## 3. 日常怎么改

改完一律 `./mihomo-proxy deploy`。`deploy` 会自己判断要不要重启，通常只热重载，不断网。

| 想做的事 | 改哪里 | 怎么改 |
|---|---|---|
| 加一个机场 | `subscriptions.list` | 加一行（见第 4 节），`deploy` |
| 换订阅 token | `subscriptions.list` | 改那一行的 URL，`deploy` |
| 删一个机场 | `subscriptions.list` | 删那行（缓存文件可留下，不影响），`deploy` |
| 某机场改为手动选节点 | `subscriptions.list` | 模式列改成 `select`，`deploy` 后在面板里点节点 |
| 加一个必须直连的域名 | `config/rules/custom_direct.yaml` | 加 `- DOMAIN-SUFFIX,example.com`，`deploy` |
| 加一个必须走代理的域名 | `config/config.yaml` 的 `rules:` | 插在 `GEOSITE,CN,DIRECT` 之前，`deploy` |
| 开「机场全挂自动换下一个」 | `config/config.yaml` 的 `rules:` | 把 `- MATCH,手动选择` 与 `# - MATCH,自动兜底` 的注释互换，`deploy` |
| 改端口 / DNS / TUN | `config/config.yaml` 静态区 | 改完 `./mihomo-proxy deploy`（`listen` 变了会自动重启） |
| 更新地理库 | `config/config.yaml` 的 `geox-url` | 见第 6 节 |

**不要手改**：`config/config.yaml` 里 `BEGIN AUTOGEN` 到 `END AUTOGEN` 之间（由清单生成）。

---

## 4. `subscriptions.list` 格式

竖线 `|` 分隔，一行一个订阅，`#` 开头是注释：

```
# key | 名字 | 订阅URL | 模式(url-test|select)
proxy1 | WD     | https://api.example.com/sub?target=clash&... | url-test
proxy2 | 传家宝 | https://sub.example.com/api/v1/client/...      | url-test
proxy3 | 赔钱   | https://another.example.com/api/v1/...         | url-test
```

| 列 | 含义 | 能否省略 |
|---|---|---|
| `key` | 决定缓存文件名（`proxies/<key>.yaml`）、节点前缀（`[P1]`）、组名（`P1-WD`）。**分配后不要再改**，改了等于换了一个新机场 | 不能 |
| `名字` | 显示用，也会进组名 | 不能 |
| `订阅URL` | Clash 格式订阅地址（面板里选 "Clash 订阅"） | 不能 |
| `模式` | `url-test`=自动挑最低延迟；`select`=手动钉死节点 | 可省略，默认 `url-test` |

格式要点：竖线两侧空格随意；URL 里不允许出现裸 `|`（合法 URL 本来也不会有）；中文和空格都行。

---

## 5. `config.yaml` 配置项速查

文件里每个参数旁边都有注释解释「是什么 / 为什么这么设 / 改错会怎样」，这里只做索引。

| 区块 | 关键项 | 作用 |
|---|---|---|
| 入口 | `mixed-port: 7890` | HTTP + SOCKS5 共用端口 |
| | `allow-lan` | 是否允许局域网其它设备用本机做代理 |
| | `ipv6` | IPv6 总开关，关掉会阻断所有 v6 连接 |
| | `log-level` | `debug` 排查用，平时 `info` |
| | `unified-delay` | 测速剔除握手耗时，跨协议/跨机场延迟才可比 |
| Web UI | `external-controller: :9090` | REST API 地址，面板靠它工作；`:9090` 无密码暴露局域网，介意就改 `127.0.0.1:9090` |
| | `external-ui: ui` | 面板静态目录，访问 `/ui` |
| 记忆 | `profile.store-selected` | 记住面板里选的节点，重启不丢 |
| | `profile.store-fake-ip` | 记住 fake-ip 发号，重启不断已有连接 |
| TUN | `tun.enable` | 建虚拟网卡，无感知接管全系统流量 |
| | `tun.stack: system` | 用内核协议栈（最快）；`gvisor` 兼容好但慢 |
| | `tun.auto-route` | 抢默认路由，否则流量进不来 |
| | `tun.auto-redirect` | 把已建立的连接也导进网卡 |
| | `tun.dns-hijack` | 劫持 53 端口，强制走内置 DNS |
| DNS | `dns.enhanced-mode: fake-ip` | 回假 IP 记账，才能按域名分流；**TUN 模式下必需** |
| | `dns.fake-ip-range` | 假 IP 网段，不能和真实内网冲突 |
| | `dns.fake-ip-filter` | 这些域名用真实解析（NTP、音乐 App、局域网） |
| | `dns.default-nameserver` | 只用于解析 DoH 服务器域名；**必须用 853（`tls://`）否则被 TUN 劫持形成死循环** |
| | `dns.nameserver` | 真实解析用的上游 |
| | `dns.nameserver-policy` | 指定域名用指定 DNS（公司内网 `uniontech.com` 走 `10.20.0.10`） |
| 订阅/分组 | `proxy-providers` / `proxy-groups` | **AUTOGEN 区，由 `subscriptions.list` 生成，别手改** |
| 规则集 | `rule-providers.custom_direct` | 挂载 `rules/custom_direct.yaml` |
| 规则 | `rules:` | 从上往下匹配，第一条命中即停 |
| 地理库 | `geo-auto-update` / `geox-url` | 库自动更新开关与地址，见第 6 节 |

### 规则表（默认）

```yaml
rules:
  - GEOIP,lan,DIRECT,no-resolve      # 内网 IP 直连
  - RULE-SET,custom_direct,DIRECT    # 自定义直连清单，优先级高于地理库
  - GEOSITE,CN,DIRECT                # 走内置 CN 域名分类库
  - GEOIP,CN,DIRECT                  # 国内 IP 直连
  - MATCH,手动选择                    # 其余走你选的机场（或 自动兜底）
```

加规则一定要插在 `GEOSITE,CN,DIRECT` 之前，否则永远轮不到。

### 策略组

```
手动选择（select）  ← 你只在这里点：选哪个机场
├─ P1-WD     url-test   ← 自动用该机场最低延迟节点
├─ P2-传家宝  url-test
├─ P3-赔钱    url-test
├─ P4-宝可梦  url-test
└─ P5-WD-2   url-test
自动兜底（fallback，hidden）  ← 启用后：当前机场全挂自动换下一个
GLOBAL（内置）                ← 只有 mode=global 才有意义
```

---

## 6. 地理库（GEOSITE / GEOIP）

| 规则 | 查什么 |
|---|---|
| `GEOSITE,CN` | `geosite.dat`：域名属于哪个分类 |
| `GEOIP,CN` | `Country.mmdb`：IP 属于哪个国家 |
| `GEOIP,lan` | mihomo 内置私有网段，不读文件 |

两个库都是二进制，编辑器打开是乱码，**不能手改**。查某个域名在不在库里：

```bash
strings config/geosite.dat | grep uniontech
```

**自动更新**：`config/config.yaml` 末尾把 `geo-auto-update: false` 改成 `true`，`deploy`。

**手动替换**：

```bash
cd config
curl -fL -o geosite.dat  "https://ghfast.top/https://github.com/MetaCubeX/meta-rules-dat/releases/latest/download/geosite.dat"
curl -fL -o Country.mmdb "https://ghfast.top/https://github.com/MetaCubeX/meta-rules-dat/releases/latest/download/country-lite.mmdb"
cd .. && ./mihomo-proxy deploy
```

（`geox-url` 用的是 `ghfast.top` 镜像，GitHub 直连不通时靠它；有直连能力就去掉前缀。）

---

## 7. `status` 输出怎么看

```
内核    mihomo v1.19.33（期望 v1.19.33，一致）
服务    active（... 启动，enabled）
源码    /home/u/project/mihomo-proxy/config
清单    /home/u/project/mihomo-proxy/subscriptions.list
漂移    无
订阅
        proxy1  P1-WD      url-test  节点 63
        ...
分组    手动选择 → P1-WD；自动兜底 未启用（组存在，规则未指向）
规则集  custom_direct：10 条
出口    直连 通 ｜ 代理 通
备份    1 份，最近 mihomo.bak.20261118-1539（回滚命令...）
```

`漂移` 不是「无」就说明源码改过还没部署（或有人动了 `/etc/mihomo`）。

---

## 8. 装到新机器

```bash
# 1. 把仓库拷过去（git clone 或 scp -r）
cd mihomo-proxy
./mihomo-proxy deploy
```

`deploy` 会依次：装内核 → 铺配置 → 接管 DNS → 起服务 → 自检。仓库里 `subscriptions.list` 的订阅地址是占位符（`xxxx`），**部署前必须换成你自己的订阅**，然后 `./mihomo-proxy deploy`。

DNS 两种模式由脚本自动判定：

| 模式 | 判定 | 链路 |
|---|---|---|
| `udcp`（UOS） | 有 udcp 包 / iptables 有 `OUTPUT_UDCP ... dport 53 DROP` | 应用 → systemd-resolved(`127.0.0.53`) → mihomo(`127.0.0.1:1053`) |
| `plain` | 其它 Linux | 应用 → mihomo(`0.0.0.0:53`)，停用 systemd-resolved |

源码里 `dns.listen` 写的是 `0.0.0.0:1053`，`deploy` 会按实际模式改写部署目录里那一行（源码不动）。

---

## 9. 排障

| 现象 | 处理 |
|---|---|
| 网页全打不开，`ping 8.8.8.8` 通但域名不通 | DNS 链路断了。`systemctl status mihomo systemd-resolved`；再 `./mihomo-proxy deploy` |
| `deploy` 报「配置校验失败」 | 没推送，线上没变。按报错修 `config/config.yaml` 或规则文件 |
| `deploy` 报「热重载失败」 | 会自动退回重启；仍失败就 `sudo systemctl restart mihomo` 看日志 |
| 改了没生效 | `./mihomo-proxy status` 看「漂移」，有漂移就再 `deploy` |
| 某机场节点数 0 | 机场按出口 IP 拦截。面板里切到别的机场，再 `deploy` 重拉 |
| 订阅返回 `dmxlc3M6...` | 给的是通用/v2ray 订阅，换 Clash 格式 |
| 面板打不开 | `curl -s http://127.0.0.1:9090/version`；不通说明服务没起 |
| 手改坏了 / 想回滚 | `./mihomo-proxy rollback`（指定快照：`rollback /etc/mihomo.bak.2026...`） |
| 服务怎么都起不来（急救） | `sudo systemctl stop mihomo && sudo rm /etc/resolv.conf && printf 'nameserver 223.5.5.5\n' \| sudo tee /etc/resolv.conf` |

日志：`journalctl -u mihomo -f`

---

## 10. 把 SKILL.md 装成 agent 技能（可选）

```bash
mkdir -p ~/.agents/skills/mihomo-proxy
ln -sf ~/project/mihomo-proxy/SKILL.md ~/.agents/skills/mihomo-proxy/SKILL.md
```

之后 AI agent 遇到「部署/更新本机代理」这类任务会自动按 `SKILL.md` 的流程调用 `./mihomo-proxy deploy`。

---

## 11. 许可

内核 MetaCubeX/mihomo (GPL-3.0)；本仓库的配置与脚本为本地维护版本。
