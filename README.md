# skland-auto-sign

森空岛自动签到 — 明日方舟 & 明日方舟：终末地（角色签到 + 登岛检票），支持多账号、token 失效自动密码续期、Server酱推送。

零依赖：纯 Python 标准库实现（HTTP/加密/gzip 全内置），任何裸 Python 3.6+ 容器直接跑，无需 pip install。

## 功能

- **明日方舟**：角色签到 + 登岛检票
- **明日方舟：终末地**：角色签到（多角色遍历）+ 登岛检票
- **多账号**：token 或账号密码，任意组合
- **自动续期**：token 失效时自动密码登录换新 token，免去每月重新抓包
- **通知推送**：Server酱（`serverchan://` / SendKey / 完整 URL 均可），正文为单行汇总模板

## 快速开始

```bash
python skland_sign.py
```

### 1. 获取 Token

三种渠道任选，拿到的都是同一个东西（鹰角通行证 token，API 返回的 msg 原文即"鹰角网络通行证账号的登录凭证"）：

| 渠道 | 步骤 |
|---|---|
| 森空岛网页版 | 登录 [skland.com](https://www.skland.com/) → 访问 `web-api.skland.com/account/info/hg` → 复制 `data.content` |
| 鹰角通行证中心（部分项目推荐） | 登录 [user.hypergryph.com](https://user.hypergryph.com/login) → 访问 `web-api.hypergryph.com/account/info/hg` → 复制 `data.content` |
| 森空岛 App | 我的 → 设置 → 鹰角通行证相关页面 → 复制 Token |

**有效期事实**（社区项目交叉结论）：约 30 天；**登出对应平台账号、修改密码会使 token 立即失效**——获取后不要在原平台点退出登录。过期后重新获取，或配置 `SKLAND_PHONE` + `SKLAND_PASSWORD` 让脚本自动续期（密码登录直连 `as.hypergryph.com` 通行证认证服务，不受任何网页会话影响）。

> **Token vs Cred**：token 是鹰角通行证级凭证（约 30 天，需自行获取）；cred 是脚本每次运行时通过 OAuth2 自动换取的短期凭证（几小时），无需关心。

### 2. 填入凭证

`creds.txt`（每行一个 token，`#` 开头为注释，已被 .gitignore 忽略）：

```
你的token粘贴在这里
```

或者不配 token，直接配手机号密码（纯密码模式）。

## 配置

全部支持环境变量，云效 / GitHub Actions 等流水线场景无需改动代码：

| 环境变量 | 说明 | 示例 |
|---|---|---|
| `SKLAND_TOKENS` | 鹰角通行证 token，逗号分隔多个 | `tok1,tok2` |
| `SKLAND_PHONE` | 手机号（密码登录 / 自动续期） | `13800000000` |
| `SKLAND_PASSWORD` | 账号密码 | `****` |
| `SKLAND_NOTIFICATION_URLS` | Server酱推送 URL，逗号分隔多个 | `serverchan://SCTxxxx` |
| `SKLAND_NOTIFY_STRICT` | `true` 时推送失败以非零码退出（流水线告警用） | `true` |

也可写在 `config.json`（`phone` / `password` / `notificationUrls` / `games` / `log_to_file`）。

### 凭证优先级

```
SKLAND_TOKENS / creds.txt 的 token
  ├─ 有效 → 直接使用（推荐日常路径，避免频繁密码登录触发风控）
  └─ 失效 → 若已配置 SKLAND_PHONE + SKLAND_PASSWORD
              → 自动密码登录换新 token → 写回 creds.txt（容器只读时跳过）→ 继续签到
```

只配密码不配 token = 纯密码模式，每次运行都密码登录。

### 通知效果

```
【明日方舟：终末地】官服角色 管** lv.55 签到成功，获得了「龙门币」500个，「合成玉」50个
【签到日历】明日方舟：终末地 本月已签 21/31 天 · 明日可得「高级作战记录」3个
【登岛检票】明日方舟：终末地 检票成功
【明日方舟：终末地】官服角色 管** lv.55 今天已经签到过了
```

昵称自动打码（首字 + `*`），角色带等级，失败行附错误码。

## 流水线部署（阿里云效等）

容器每次运行都是全新环境，文件持久化无意义，全部走环境变量：

1. 流水线添加环境变量 `SKLAND_TOKENS` + `SKLAND_NOTIFICATION_URLS`（日常）
2. 加 `SKLAND_PHONE` + `SKLAND_PASSWORD`（token 月抛时自动续命，当次运行直接完成签到）
3. 定时触发器设为每日一次即可

## 认证链路

```
鹰角通行证 token
  → POST as.hypergryph.com/user/oauth2/v2/grant          （grant_code，一次性）
  → POST zonai.skland.com/web/v1/user/auth/generate_cred_by_code
  → cred + sign_token
  → sign = MD5( HMAC-SHA256( sign_token, path + body + timestamp + headerCA_json ) )
```

`generate_cred_by_code` 自 2024-09 起强制校验数美设备指纹（机房 IP 尤其严格），脚本内置纯 Python 数美实现自动注册真实 dId，无需浏览器、零编译依赖。

登岛检票独立于角色签到：`POST /api/v1/score/checkin`，body `{"gameId": "1"|"3"}`（方舟=1，终末地=3），每游戏每日各一次。

## 响应码

| code | 含义 |
|---|---|
| `0` | 成功 |
| `10001` | 今日已签到（正常响应） |
| `10003` | 签名校验失败（本地时钟偏差 >30s，脚本启动时已自动校准） |

## 致谢

- [xjwwjx/skland-auto-sign](https://github.com/xjwwjx/skland-auto-sign) — 签名实现与纯 Python 密码学
- [YueHen14/skyland-auto-sign](https://github.com/YueHen14/skyland-auto-sign)（[FancyCabbage](https://gitee.com/FancyCabbage/skyland-auto-sign)）— 密码登录
