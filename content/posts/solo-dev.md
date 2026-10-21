---
title: "自托管的死人开关：一个 solo dev 的密钥交接工程"
date: 2026-10-20T17:47:14+08:00
draft: false
tags: ["dead-mans-switch", "age", "shamir", "secrets-management", "systemd"]
categories: ["工程实践"]
description: "用 systemd timer、外部心跳、age 信封加密和 Shamir 分片，把「我不在了」做成一套能演练、能撤回、能轮换的存取策略。"
---

先回答一个十秒就能答完的问题：那个能读写生产数据库的 token、那个付着服务器账单的云账号、那个管着域名的注册商后台 —— 现在有几个独立的人或介质，能在你明天起不来的情况下拿到它？

如果答案是零，或者只是「我对象知道我的电脑密码」，那你手上不是情感问题，是一个存取策略问题。你目前把所有凭据的读取权，绑在了「你还活着，并且还记得住」这一个条件上。这个条件的可用性，比你以为的低得多。

本文不谈「人生无常」。下面每一节都是可验证的工程判断：三种失败模式、判定层、密文层、分片层，最后是一次能今晚就跑的演练。所有版本号、默认值、实测字节数都标注了来源；查不到的我会直说查不到。

## 一、三种死法：先定义失败，再谈方案

**结论先说：死人开关的验收标准不是「出事时能发出一封邮件」，而是三条判据同时成立 —— 误触发可撤回、它自己坏掉能被发现、单独一个持有人打不开门。**

第一种死法是误触发。你在外地，手机丢了，连续两周没点确认链接，系统按设计把生产 root 凭据发给了还在跟你打诉讼的前合伙人。这类系统里做得最保守的商用服务，是把提醒切成多级：deadmansswitch.net 在最后一次露面后的第 30、45、52 天分别提醒你，第 60 天才真正把邮件发出去。为什么重要：交接动作一旦执行往往不可逆，收不回来的不是文件，是访问权。「多级提醒 + 足够长的宽限」不是洁癖，是这类业务的默认值。

第二种死法是静默失效。这是三种里最阴的一种，因为你在活着的时候完全看不见它。systemd 的手册写得很清楚：`OnCalendar=` 走的是墙上时钟（realtime），而 `OnActiveSec=`、`OnUnitActiveSec=` 这一类走的是单调时钟（monotonic），「如果机器被临时挂起，单调时钟一般也会暂停」。再加上 `Persistent=` 默认是 `false`，并且只对 `OnCalendar=` 生效 —— 也就是说，用单调计时器加上一次容器重启（或者一次挂起），你的心跳就断在没有补跑机制的地方。你看到的界面是「timer 已 enable」，实际发生的事是「半年没发过一次心跳」。为什么重要：这类失败只有第二个系统能发现，而大多数人的开关是单系统。

第三种死法是单点信任。开关的维护者自己能解开全部内容，于是「交接」变成了「把命交给另一个人」。这不是假想：`vault operator init` 的默认参数就是 `-key-shares=5`、`-key-threshold=3` —— 五片、要三片才能开，这本身就是工业界对「开门不该由一个人决定」的回答。

想找一个干净的公开案例来镇场子，会失望。规模最大的公开记录是 QuadrigaCX：创始人 Gerald Cotten 于 2018 年 12 月去世，最多 C$2.5 亿（约 US$1.9 亿）客户资产「失踪，或者因为只有 Cotten 掌握离线冷钱包的密码而无法访问」。但这案子必须带上后半句：2020 年 6 月安大略证券委员会（OSC）的结论是，该公司本身就是欺诈，原话是「发生在 Quadriga 的事，是用现代技术包装的老式诈骗」。它只能当「单点持有、人一死就全停摆」的公开例子，不能拿来当「密钥丢失」的技术证据。更值得警惕的是另一件事：除了它，我找不到一条技术事实清楚、可公开引用的 solo dev 密钥丢失记录 —— 公开可查的同类记录极少，这本身就是坏信号，因为没人写复盘往往意味着当事人连写复盘的能力都没有了。

把三种死法翻成可检的判断，就四条：

- 触发器误报时，损失可逆吗？
- 心跳静默失效时，你会不会以为开关还在工作？
- 任何一个持份人单独行动，能拿到明文吗？
- 密钥在你活着的时候轮换过一次，流程能跟着轮换走吗？

四条塞进一张表，就能看出市面上的方案各自卡在哪：

| 方案 | 误报可撤回 | 自身坏了知道吗 | 几人能开门 |
|---|---|---|---|
| deadmansswitch.net | 是（30/45/52 天提醒，60 天才发） | 未知（没有「监控监控者」的文档） | 1（服务方持有全部明文） |
| Posthumous | 是（WARNING/GRACE 阶段可打卡回到 ARMED） | 是（联邦节点互相健康检查，`peer_down_threshold` 默认 6 小时） | 无门限（只发通知、跑脚本） |
| Shardium | 是（90 天邮件 ping） | 否（依赖托管服务存活） | 2-of-3，固定 |
| Vault + Shamir | 是（未 unseal 前没有损失） | 部分（seal 状态可查） | 3-of-5，默认 |

这一步最容易错在哪：把「通知渠道」当主线。发邮件是最简单的一环，也是最没用的一环 —— 通知到了，不代表收件人拿得到能用的东西。

## 二、判定层：怎么「证明」你死了

**结论先说：判定层的输出应该是证据强度，不是一条告警。任何单一通道判定为「死」，都不足以触发交接。**

触发器只有两类可信来源，而且它们信任的东西正好相反。本机的 cron 和 systemd timer 信任的是你自己的机器 —— 好处是时间源在你手里，坏处是机器一关机、一挂起、一换容器，你就失去了观测能力。外部 ping 服务（healthchecks.io 这类）信任的是第三方 —— 好处是整台机器消失时它还在说话，坏处是它的告警语义和额度你要自己看清楚。两者不是替代关系，是互补关系：至少两条独立通道，任一条为「死」都不触发。

第二件事是把「宽限期（grace period）」和「连续 N 次未心跳」分开。宽限期防抖：某次定时任务晚了二十分钟不该惊动任何人。连续 N 次防的是单点故障：一次没发出去可能是网络抖动，连续三次就是系统性问题。两者都要有，用一个代替另一个都会留下可预测的漏洞。顺便说清一个不确定项：healthchecks.io 的文档解释了 Grace Time 是「检查迟到后，再等多久才告警」，但没给默认值和上限 —— 我查了配置文档、定价页和 FAQ 都没有数字。所以这里只能给建议值：把宽限期设成 `Period` 的 1–2 倍，这是建议，不是官方默认值。

第三件事是时间源，它既是攻击面也是事故源。systemd 手册里有几个数字值得背下来：`AccuracySec=` 默认 1 分钟（目的是省电、合并唤醒，不是给你做抖动）；`RandomizedDelaySec=` 默认 0（用于把同类任务摊开，防尖峰）；睡眠期间多次到点的 calendar timer，恢复后「只会产生一次服务激活」；`Persistent=true` 的补跑也只是立刻触发一次。为什么重要：这意味着用 calendar 加 Persistent 不会「一次性触发一串动作」，但会制造另一种错觉 —— 你以为它每天都在跑，其实它在恢复后只跑了一次。如果你用的是 healthchecks.io 的 Cron 模式，还要填对 Server's Time Zone，官方明确要求，因为 cron 守护进程通常用本机时区；时区填错就是系统性误报。

第四个现实约束是档位。想写「每 30 天一次」的长周期心跳，先看你手里的工具支不支持：Uptime Kuma 原生支持 Push 类型的心跳监控，但它的 issue #7264 就是在请求「允许 30 天以上的心跳间隔」，说明当前界面上限到不了 30 天。长周期交接不能挂在「默认档」上。

下面这套是最小可用的本机心跳，systemd 262 下可以直接跑（把 `<PING_URL>` 换成你 check 页面里给出的 ping 地址；我不替你编造服务端 URL）：

```ini
# /etc/systemd/system/deadman-heartbeat.service
[Unit]
Description=Dead-man switch heartbeat
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
# PING_URL 从 check 的配置页复制；-f 让 4xx/5xx 也变成失败，failure 在 journal 里可见
ExecStart=/usr/bin/curl -fsS -m 10 --retry 3 -o /dev/null "<PING_URL>"
```

```ini
# /etc/systemd/system/deadman-heartbeat.timer
[Unit]
Description=Hourly dead-man switch heartbeat

[Timer]
OnCalendar=hourly
Persistent=true
AccuracySec=1min
RandomizedDelaySec=5min
Unit=deadman-heartbeat.service

[Install]
WantedBy=timers.target
```

```bash
# systemd 262
sudo systemctl daemon-reload
sudo systemctl enable --now deadman-heartbeat.timer
systemctl list-timers deadman-heartbeat.timer --all
journalctl -u deadman-heartbeat.service -n 20 --no-pager
```

注意这里刻意用了 `OnCalendar=` 加 `Persistent=true`：补跑只发生一次，而且补跑这件事本身会被 journal 记下来，你事后能看见「它睡了一周，醒了补了一次」。同时 `curl -f` 让 ping 返回 4xx/5xx 时 service 直接失败，而不是安静退出 —— 心跳发送失败必须是一条可查询的失败记录，而不是一段沉默。

这一步最容易错在哪：把「告警发出去了」当成「证据成立」，然后把宽限期当保险，省掉连续 N 次的判定；或者反过来，只用一条通道，让机器挂起的那半分钟直接演成一次交接。

## 三、密文层：把「信封」做对

**结论先说：密文层的目标不是「把文件加密」，是「轮换的时候只换信封」。做不到这一点，你的交接流程会在第一次轮换后变成一堆打开不了的垃圾。**

信封加密（envelope encryption）的正确形状只有三句话：数据密钥加密内容；数据密钥被每个收件人的公钥各封一份；轮换只重封信封，不动内容。age 就是围绕这个形状设计的 —— 每个文件一个 16 字节（128 位）的 CSPRNG file key，禁止跨文件复用；X25519 收件人那一段的封装是「临时 X25519 交换 → HKDF-SHA-256 → ChaCha20-Poly1305」，nonce 固定为 12 个 0x00 字节；载荷按 64 KiB 分块用 ChaCha20-Poly1305 加密；header 完整性用 HMAC-SHA-256；口令模式（`--passphrase`）走 scrypt。这些不是我凭记忆写的，是对着 C2SP 的 age 规范 v1.1.0 抄的。

为什么重要：轮换是这类系统唯一的高频运维动作。你的收件人可能会换（换律师、换合伙人、换掉一个不再信任的人），收件人的密钥也会换。如果加密方式绑死在内容上，每次换人都得重新处理全部密文；而信封加密让你只重封那一层几十字节的东西。

用实测数字校准一下感觉。age v1.3.2 加密 59 字节明文，得到的文件是 259 字节：header 168 字节、加密开销 32 字节、载荷 59 字节。也就是说信封本身的开销是固定的 200 字节上下，跟内容大小无关 —— 这正是「只换信封」可行的物理基础。顺带说，v1.3.0 起 age 引入了后量子收件人类型 `mlkem768x25519`（recipient 前缀 `age1pq`，identity 前缀 `AGE-SECRET-KEY-PQ-`），并新增了 `age-inspect`；默认的 `-r` 收件人仍然是 X25519，`age-inspect` 会明确告诉你「does NOT use post-quantum encryption」。这个差别值得知道，但不必现在就切换。

age 和 sops 的分工别搞混：age 是文件级加密原语，收件人就是公钥；sops 是「文件里只加密 value、metadata 留明文、diff 还能看」的工作流。sops v3.13.3 的官方 README 承诺的后端是 AWS KMS、GCP KMS、Azure Key Vault、HuaweiCloud KMS、age 和 PGP 六类 —— 请注意这个清单里没有 Vault，别替它补，Vault 那条路是 keyservice 或者自己写。

反模式要说一句：把 root token 塞进一个带密码的 zip，再把 zip commit 到私有仓库。那个 zip 的密码存在哪儿？如果答案还是「在仓库的 README 里」，你只是把明文换成了明文加一层心理安慰。「加密的密码」必须有独立的存放位置，否则它不叫加密。

硬件因子（YubiKey、age 插件）提升的是「抢到文件也解不开」这一层，不解决「持有人失联」那一层。`age-plugin-yubikey` 目前是 v0.5.1（2026-04-08 发布，main 最后提交 2026-04-24），仍在维护但节奏很慢，大约半年一次，而且只支持 YubiKey 4/5 系（PIV）。把它当「多加一道门」，不要当「交接方案」。

```bash
# age v1.3.2（官方 release 的 windows-amd64 / linux-amd64 均可用）
age-keygen -o key.txt
# 输出形如：Public key: age1...      ← 这行就是收件人，写进交接文档
# 注意 age-keygen 会警告：私钥被写进了 world-readable 的文件，先 chmod 600

# 收件人直接从私钥文件的注释行里取，不靠人肉抄写
RECIPIENT=$(grep -oE 'age1[0-9a-z]+' key.txt | head -n 1)

printf 'root-token: <你的真实凭据，别用这个占位符>\n' > secrets.txt
age -r "$RECIPIENT" -o secrets.age secrets.txt

# 自检：确认收件人类型与体积，别只看「命令没报错」
age-inspect secrets.age
# 期望看到 recipient types: - "X25519"

# 演练时必须真的解一次，不能只加密不验证
age -d -i key.txt secrets.age
```

这一步最容易错在哪：把口令模式当默认（scrypt 再硬，口令还是人能猜的东西，且它让「多人各持一份」这件事无法实现），以及把私钥和密文放进同一个备份里 —— 那样分片分的是同一份东西。

## 四、分片层：让「开门」需要不止一个人

**结论先说：门限不是「把东西切成几份放几个地方」，是「少于 k 份等于零信息，并且恢复结果必须能验证」。少了后半句，你会拿一堆垃圾当明文。**

Shamir 门限分享（Shamir's Secret Sharing）的直觉是：构造一个 k-1 次多项式，把秘密放在常数项上，n 个分享是曲线上的 n 个点。任意 k 个点能唯一确定这条曲线，因此能还原常数项；少于 k 个点时，这条曲线有无限多种可能，信息量为零。Vault 把它用作 unseal 机制，默认 5 片、门限 3；而 `-recovery-shares` / `-recovery-threshold` 默认同样是 5/3，但只在 Auto Unseal（HSM、KMS、Transit 这类 seal）场景下可用。这里有个必须说清的区分，否则轮换时会踩：**unseal key 解的是 root key，能开库；recovery key 不能解密 root key，因此在自动解封机制失效时不足以 unseal**。Vault 官方还有一句更狠的话：如果 seal 机制或它的密钥被永久删除，集群就恢复不了，即使你有备份。

如果你想要的是「助记词形态的分片」，权威方案是 SLIP-0039（状态 Final，2017-12-18 创建，标题就是 Shamir's Secret-Sharing for Mnemonic Codes）。它是两层级结构（group / member），在 GF(256) 上逐字节做 Shamir，单组上限 N ≤ 16、T ≤ 16，分享是 20 或 33 个词的助记词。Trezor 官方支持 Shamir backup，并且支持跨会话、分批恢复 —— 这一条对「分片在不同人手里、不同时间才凑齐」的真实场景很关键。

下面这个 3-of-5 的实现是纯标准库 Python，本机跑过，输出是真的（3 片还原成功，2 片还原出错误内容）：

```python
"""Shamir's Secret Sharing over GF(256) — stdlib only (3-of-5 demo).
Run: python shard.py
"""
import secrets
import sys

# GF(256) with AES polynomial 0x11b
EXP = [0] * 512
LOG = [0] * 256
x = 1
for i in range(255):
    EXP[i] = x
    LOG[x] = i
    x ^= (x << 1) ^ (0x11B if x & 0x80 else 0)
    x &= 0xFF
for i in range(255, 512):
    EXP[i] = EXP[i - 255]


def mul(a, b):
    if a == 0 or b == 0:
        return 0
    return EXP[LOG[a] + LOG[b]]


def div(a, b):
    if b == 0:
        raise ZeroDivisionError
    if a == 0:
        return 0
    return EXP[(LOG[a] - LOG[b]) % 255]


def eval_poly(coeffs, x):
    acc = 0
    for c in reversed(coeffs):
        acc = mul(acc, x) ^ c
    return acc


def split(secret, k, n):
    if not (1 <= k <= n <= 255):
        raise ValueError("need 1 <= k <= n <= 255")
    shares = [(i, bytearray(len(secret))) for i in range(1, n + 1)]
    for pos, byte in enumerate(secret):
        coeffs = [byte] + [secrets.randbelow(256) for _ in range(k - 1)]
        for sx, buf in shares:
            buf[pos] = eval_poly(coeffs, sx)
    return [(sx, bytes(buf)) for sx, buf in shares]


def combine(shares):
    k = len(shares)
    out = bytearray(len(shares[0][1]))
    for pos in range(len(out)):
        acc = 0
        for j in range(k):
            xj, yj = shares[j][0], shares[j][1][pos]
            num, den = 1, 1
            for m in range(k):
                if m == j:
                    continue
                xm = shares[m][0]
                num = mul(num, xm)
                den = mul(den, xj ^ xm)
            acc ^= mul(yj, div(num, den))
        out[pos] = acc
    return bytes(out)


if __name__ == "__main__":
    secret = b"root-token: PLACEHOLDER-ROTATE-ME"
    shares = split(secret, 3, 5)
    print("secret :", secret.decode())
    for sx, data in shares:
        print(f"share {sx}: {data.hex()}")
    r3 = combine([shares[0], shares[2], shares[4]])
    r2 = combine([shares[0], shares[1]])
    print("3 shares ->", r3.decode(), "| match:", r3 == secret)
    print("2 shares ->", r2.hex()[:32], "... | match:", r2 == secret)
    sys.exit(0 if r3 == secret and r2 != secret else 1)
```

实测输出（本机 Windows 11 + Python 3.14，`python shard.py`，退出码 0；这里只留前两片，share 3–5 略）：

```text
secret : root-token: PLACEHOLDER-ROTATE-ME
share 1: 8699fc805e975ca0fd1d3a0230a428719c5c68cc0be0170ef30539fa6c535b4a9b
share 2: e7c436d1497baf6f346794fa1e2cee63b285d1098e302c5bae3bb5cfeecc0f1665
3 shares -> root-token: PLACEHOLDER-ROTATE-ME | match: True
2 shares -> 505bba46533a0de5bac2a9a32adc6a7f ... | match: False
```

请看最后一行：2 片并不是「报错」，而是安静地还原出一串错误内容。这就是为什么 SLIP-0039 要在规范里用 `f(254)` 存一个 digest，专门用来验证恢复是否正确。**没有校验的分片方案，失败方式是给你一份看起来像凭据的垃圾**，而你在出事那天没有第二次机会去发现自己拿错了。

分片的实务坑不在密码学，在物理世界：异介质、异地、异人。同一台机器上的三个文件不是三片，是一片；同一个云账号下的三个对象存储也不是三片。分片交给谁，各自的法律效力和响应速度完全不同：律师和会计走合同，合伙人走信任，第二个云账号走操作习惯，纸质保险箱走物理安全。这里还有几条现成的现实约束：Bitwarden 的 Emergency Access 是付费功能（premium，含付费组织成员），它的 Takeover 会**替换**你的主密码并清除两步登录方式；1Password 的官方答案干脆不是自动放行，而是一张 Emergency Kit PDF —— 里面是 sign-in address、邮箱、Secret Key 和手写密码的空位，建议你打印出来放进保险箱。自托管的 Vaultwarden 实现了 Emergency Access，但仍要跟着官方协议改（2026-10-06 合并的 PR #7746 就是在补新的密码载荷），社区里也有「发不起 emergency access 请求」的报告。

还有一条容易被忽略的法律面：如果持份人会接触到含个人数据的系统或凭据，他的身份在 GDPR 下是 processor，你必须和他签符合 Art. 28(3) 的合同（按书面指示处理、保密、采取 Art. 32 措施、协助响应数据主体权利、终止时删除或返还、配合审计）；如果他自行决定处理目的与手段，就按控制者论处（Art. 28(10)）。中文语境下对应《个人信息保护法》第 21 条的委托处理义务，结构类似，但我没有逐条核对。这不是「要不要相信他」的问题，是「出事之后你能拿出什么」的问题。

这一步最容易错在哪：在同一台机器上做「三份备份」，以及把恢复路径做得比加密路径还复杂 —— 出事那天接手的人通常悲伤、慌乱，而且不懂你的技术栈。

## 五、演练：没有 dry-run 的死人开关等于没有

**结论先说：没演练过的开关是一种信念，不是一套系统。而这类系统最贵的一次失败，恰好是它第一次真正运行的时候。**

一次完整演练要覆盖六步，缺一步都算没演练：模拟 N 天没有心跳 → 触发 → 收件人用自己手里的东西解密 → 拿到能用的凭据 → 撤销并回到正常态 → 确认撤销生效。注意第三步：必须是「收件人视角」，不是你本人视角。你自己解一次不算数，因为你知道所有东西在哪儿；这正是交接最常见的失败点。

「监控监控者」有现成范式可抄。Posthumous 用联邦式多节点互相检查，任一节点可触发，作者自己写明的失败模式是「重复通知，而不是静默」；参数上 `peer_check_interval` 默认 30 分钟、`peer_down_threshold` 默认 6 小时。请注意它明确不做什么：它的状态机是三阶段的（默认 `checkin_interval` 7 天、8 天进 WARNING、12 天进 GRACE、14 天 TRIGGERED，且 TRIGGERED 不可逆），触发之后是发通知（Apprise 支持 80 多个渠道）或跑脚本 —— **它的功能列表里没有任何密钥加密或分片能力**。Shardium 则把分片做了，但固定 2-of-3、恢复依赖托管服务存活、也不处理轮换后的分片同步。这两条正好说明一件事：市面上大多数「死人开关」止步于通知，凭据怎么安全地交出去，仍然是你自己的工程。

轮换同步是演练里最容易漏的一项。Vault 的文档明确 recovery key 与 unseal key 是两套东西，`vault operator rekey -target=recovery` 只重切其中一套 —— 「换了一套、另一套没跟」是机制上就存在的失误面。对自建方案同理：你换了 root token，密文要重封，分片要重切，收件人手里的说明要重发，演练记录要更新。四处有一处没跟上，实战那天拿到的就是过期凭据。

最后交一份一页纸的 runbook，人可读、脱离代码仓库也能拿到。演练脚本本身可以长这样（age v1.3.2 + 上面那个 `shard.py`，两者都在 PATH 里）：

```bash
#!/usr/bin/env bash
# drill.sh — 一次最小演练：分片校验 → 加密 → 自检 → 解密 → 比对
# 依赖：age v1.3.2（含 age-inspect）、python3、同目录的 shard.py
set -euo pipefail

SHARD=${1:-./shard.py}

# 1) 分片：先跑，免得 cd 到临时目录后相对路径失效
#    shard.py 的退出码已经编码了判据：3 片成功「且」2 片必须不等于原文，才返回 0
python "$SHARD" > /dev/null && echo "shamir 3-of-5: OK (2 shares correctly rejected)"

WORK=$(mktemp -d)
trap 'rm -rf "$WORK"' EXIT
cd "$WORK"

# 2) 信封：加密 + 自检 + 解密比对，三步缺一不可
age-keygen -o key.txt 2>/dev/null           # key.txt 的 "# public key:" 那行就是收件人
RECIPIENT=$(grep -oE 'age1[0-9a-z]+' key.txt | head -n 1)
printf 'root-token: PLACEHOLDER-ROTATE-ME\n' > secrets.txt

age -r "$RECIPIENT" -o secrets.age secrets.txt
age-inspect secrets.age | grep -q 'X25519' && echo "recipient type: X25519"

age -d -i key.txt secrets.age > got.txt
cmp -s secrets.txt got.txt && echo "decrypt: OK" || { echo "decrypt: FAIL"; exit 1; }
```

这段脚本里最容易被人抄错的不是 `age -d`，是收件人从哪来：`age-keygen` 会把公钥写进私钥文件的注释行，`grep -oE 'age1[0-9a-z]+'` 把它抠出来，演练就不依赖你手工抄过公钥。同理，凡是需要「人肉记住」的步骤，都是未来会断的步骤。

这一步最容易错在哪：只验证「加密成功」，不验证「收件人能在不知道任何上下文的情况下恢复」；以及演练一次就归档，之后轮换、换人、换机器都没再跑过。

## 今晚 30 分钟能做完的第一步

不要从「设计一套完美的交接体系」开始。今晚只做三件小事，全部可以在半小时内完成：

第一，生成一对 age 密钥，把一份真实凭据加密成 `secrets.age`，然后**立刻解一次**确认能解开。这一步的产出不是密文，是你知道自己有一条能用的加密路径。

第二，在心跳服务上建一个 check，把 ping 地址塞进上面那个 systemd timer（或者一条 cron），跑通一次，看 journal 里出现一条成功记录。然后故意把 Ping URL 改错一位，确认它会在 journal 里变成 failed —— 你要亲眼看见「它坏了会说话」。

第三，把私钥分两处放：一处纸质、一处跟你的密文分开的离线位置。这两处不要是同一台机器、同一个云账号、同一个房间。

做完这三件事，你的开关从「信念」变成了「一条能验证的链路」。剩下的分片、门限、法律合同、轮换，都是在这条链路上加层 —— 但加层的顺序不能反，先把一条能自证的链路跑通，比先画一张完整的架构图更值钱。

## 参考

- HashiCorp Vault，`vault operator init` 参数与默认值：https://developer.hashicorp.com/vault/docs/commands/operator/init
- HashiCorp Vault，seal / unseal key 与 recovery key 的区别：https://developer.hashicorp.com/vault/docs/concepts/seal
- HashiCorp Vault releases（v2.2.0，2026-10-15）：https://github.com/hashicorp/vault/releases
- age 规范（C2SP，v1.1.0）：https://c2sp.org/age
- age releases（v1.3.2）：https://github.com/FiloSottile/age/releases
- age-plugin-yubikey releases（v0.5.1）：https://github.com/str4d/age-plugin-yubikey/releases
- sops releases（v3.13.3）与 README 后端清单：https://github.com/getsops/sops/releases ；https://raw.githubusercontent.com/getsops/sops/main/README.rst
- SLIP-0039 规范：https://github.com/satoshilabs/slips/blob/master/slip-0039.md
- Trezor，Shamir backup 说明：https://trezor.io/learn/advanced/standards-proposals/what-is-shamir-backup
- healthchecks.io，check 配置语义：https://healthchecks.io/docs/configuring_checks/
- healthchecks.io 定价（Hobbyist 20 jobs）：https://healthchecks.io/pricing/
- Uptime Kuma releases（2.5.6）：https://github.com/louislam/uptime-kuma/releases
- Uptime Kuma issue #7264（心跳间隔上限）：https://github.com/louislam/uptime-kuma/issues/7264
- systemd.timer 手册（systemd 262）：https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html
- Bitwarden Emergency Access：https://bitwarden.com/help/emergency-access/
- Vaultwarden PR #7746（Emergency Access 新载荷）：https://github.com/dani-garcia/vaultwarden/pull/7746
- Vaultwarden discussions #3590（客户端兼容问题）：https://github.com/dani-garcia/vaultwarden/discussions/3590
- 1Password Emergency Kit：https://support.1password.com/emergency-kit/
- deadmansswitch.net：https://deadmansswitch.net/
- Google Inactive Account Manager：https://support.google.com/accounts/answer/3036546
- QuadrigaCX 案件（含 OSC 欺诈结论）：https://en.wikipedia.org/wiki/Quadriga_(cryptocurrency_exchange)
- GDPR Art. 28：https://gdpr-info.eu/art-28-gdpr/
- Posthumous 项目页：https://metafunctor.com/projects/posthumous/
- Shardium（Show HN 讨论）：https://news.ycombinator.com/item?id=46426857
- ssss（B. Poettering）：http://point-at-infinity.org/ssss/
