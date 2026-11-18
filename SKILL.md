---
name: mihomo-proxy
description: 部署或更新本机 mihomo（Clash.Meta）代理，或排查其状态时使用。只调用 ./mihomo-proxy，绝不手改 /etc/mihomo。
---

# mihomo-proxy：标准部署流程

## 适用场景

- 在新机器上首次部署 mihomo 代理
- 用户改了 `subscriptions.list` / `config/` 里的文件，需要让改动生效
- 代理异常，需要确认内核、服务、DNS、分流是否正常
- 需要卸载 mihomo 并回滚系统 DNS

## 前置条件

| 检查 | 要求 |
|---|---|
| 项目路径 | 含 `mihomo-proxy`、`subscriptions.list`、`config/` 的目录（通常 `~/project/mihomo-proxy`） |
| 系统 | Debian 系（UOS/Deepin/Ubuntu），x86_64，CPU 支持 AVX2 |
| 权限 | 普通用户即可；脚本在需要时自己调用 sudo |
| 网络 | 首次装内核若 `pkg/` 无 deb 需能访问 GitHub |

## 铁律

1. **只在项目目录内改文件**：`subscriptions.list`、`config/config.yaml`、`config/rules/*.yaml`。
2. **绝不编辑 `/etc/mihomo`**，那是部署产物，下次 `deploy` 会被覆盖。
3. **绝不编辑 `config/config.yaml` 里 `BEGIN AUTOGEN` 与 `END AUTOGEN` 之间的内容**，那由 `subscriptions.list` 生成。
4. 不手工敲 `dpkg`、`systemctl`、`ip`、`iptables` 等命令绕过脚本——除非用户明确要求排查。

## 标准流程

```bash
cd <项目目录>

# 1. 记录基线
./mihomo-proxy status

# 2. 执行部署/更新（约 10 秒；会按需提权）
./mihomo-proxy deploy
```

### 判定成功

| 信号 | 含义 |
|---|---|
| 退出码 `0` | 全部完成 |
| 自检 8 项全 `[OK]` | 服务/内核/API/UI/TUN/DNS/代理/自定义规则集 正常 |
| 输出「热重载完成（未断流）」 | 只改了配置，未重启，用户连接没断 |
| 输出「服务已重启」 | 内核或 `listen` 变了，属正常 |
| 「源码与部署目录一致，无需推送」 | 本次没有改动，属正常 |

成功后再跑一次 `./mihomo-proxy status`，确认「漂移：无」，并把订阅节点数、出口状态复述给用户。

### 退出码与失败处理

| 退出码 / 输出 | 含义 | 动作 |
|---|---|---|
| `1` | 前置检查或 `mihomo -t` 校验失败 | **线上没有被改动**。按报错定位：语法错在 `config/config.yaml`，规则错在 `config/rules/*.yaml`，清单格式错在 `subscriptions.list`。改完重新 `deploy` |
| `2` | 推送完成但自检失败 | 读 `[FAIL]` 行；DNS 项失败先用 `./mihomo-proxy deploy` 再试一次，仍失败则按 README 第 9 节排障 |
| 「热重载失败」 | API 不通或配置被拒 | 脚本会自动退回重启；若仍失败，`sudo systemctl status mihomo` 看日志 |
| 「部署后的配置校验失败，已自动回滚」 | 同步进 `/etc/mihomo` 的树有问题（常见：源码里地理库/ui 缺失） | 线上已回滚、服务未受影响。修源码后重新 `deploy` |
| 「内核安装后版本仍是 …」 | deb 与系统不匹配 | 让用户把正确的 `mihomo-linux-amd64-v3-*.deb` 放进 `pkg/` |
| 「部署目录 … 不存在」 | 从未部署过且写入失败 | 用 `sudo ./mihomo-proxy deploy` |

## 其余命令

```bash
./mihomo-proxy restart     # 先校验，通过才强制重启（改 TUN/端口后用）
./mihomo-proxy status      # 只读；任何时候都可以跑
./mihomo-proxy rollback    # 回滚到最近快照（或 rollback /etc/mihomo.bak.2026...）
./mihomo-proxy remove      # 卸载：打印清单 → 需用户输入 yes → 停服务/删包/删配置/回滚 DNS
```

`remove` 是破坏性操作，**必须先向用户确认**，再把脚本打印的删除清单转述一遍。

## 常见任务

| 用户诉求 | 操作 |
|---|---|
| 加一个机场 | 在 `subscriptions.list` 追加一行 `proxyN \| 名字 \| URL \| url-test`，然后 `deploy` |
| 换订阅 token | 改 `subscriptions.list` 里那一行的 URL（key 不变），`deploy` |
| 某机场改为手动选节点 | 模式列改 `select`，`deploy`，再到面板点节点 |
| 加必须直连的域名 | `config/rules/custom_direct.yaml` 加 `- DOMAIN-SUFFIX,example.com`，`deploy` |
| 加必须走代理的域名 | `config/config.yaml` 的 `rules:` 里插在 `GEOSITE,CN,DIRECT` 之前，`deploy` |
| 整体检查是否同步 | `./mihomo-proxy status` 看「漂移」 |
| 回滚到上次部署 | `./mihomo-proxy rollback`（需要指定就 `rollback /etc/mihomo.bak.2026...`，`status` 里有最近一份的路径） |

## 验证清单（每次部署后）

```bash
./mihomo-proxy status                                    # 漂移：无
curl -s -o /dev/null -w '%{http_code}\n' http://www.baidu.com                        # 直连应为 200
curl -s -o /dev/null -w '%{http_code}\n' -x http://127.0.0.1:7890 https://www.google.com/generate_204   # 代理应为 204
```

## 不要做的事

- 不要在 `/etc/mihomo` 里新建或修改任何文件（除排查用只读查询）
- 不要手动删除 `/etc/mihomo/cache.db`（会丢用户在面板里选好的节点）
- 不要跳过 `deploy` 直接 `systemctl restart`：那样 `/etc/mihomo` 可能还是旧配置
- 不要在 `mihomo-proxy deploy` 校验失败后强行重启服务
