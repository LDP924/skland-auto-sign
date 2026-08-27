# 开发经验笔记

> 记录开发森空岛自动签到脚本过程中踩过的坑、解决的技术难点和积累的经验。

---

## 1. 森空岛 API 签名算法

### 签名流程

```
sign = MD5( HMAC-SHA256( sign_token, path + body + timestamp + headerCA_json ) )
```

其中：
- `path`：请求 URL 的路径部分（不含域名和 query string）
- `body`：请求体的 JSON 字符串（无空格），GET 请求为空字符串
- `timestamp`：当前 Unix 时间戳（脚本中减 2 秒防时钟偏移）
- `headerCA`：包含 `platform`、`timestamp`、`dId`、`vName` 的 JSON 对象

### 关键教训

**`requests.post(data=serialized_bytes)` vs `json=`**

这是最容易踩的坑。签名计算使用的是手动序列化后的 JSON 字符串（无空格），如果用 `requests.post(json=body)`，requests 库会重新序列化 body，可能导致格式与签名不一致，签名验证失败。

正确做法：
```python
body_str = json.dumps(body, separators=(",", ":"))  # 无空格序列化
sign = compute_sign(sign_token, path, body_str)
requests.post(url, data=body_str.encode("utf-8"), headers=headers)
```

错误做法：
```python
sign = compute_sign(sign_token, path, json.dumps(body))
requests.post(url, json=body, headers=headers)  # requests 会重新序列化！
```

---

## 2. 明日方舟 vs 终末地：两套完全不同的 API

这是本项目最大的技术难点。初版脚本只支持明日方舟，终末地角色签到一直返回 `code=10001`，一度以为是正常现象（已签到），但实际上是接口用错了。

### 差异对比

| 参数 | 明日方舟 | 终末地 |
|------|---------|--------|
| 角色签到端点 | `/api/v1/game/attendance` | `/api/v1/game/endfield/attendance` |
| 请求体 | `{"uid":"...", "gameId":"..."}` | 空（无 body） |
| 额外 Header | 无 | `sk-game-role: 3_{roleId}_{serverId}` |
| 签名 platform | `"1"` | `"3"` |
| 签名 dId | 随机 UUID (base64) | `""`（空字符串） |
| 签名 vName | `"1.5.1"` | `"1.0.0"` |
| User-Agent 版本 | `Skland/1.5.1` | 同左（共用） |

### 关键发现

1. **终末地的 roleId / serverId 来源**：不是绑定列表里的 `uid` / `channelMasterId`，而是绑定列表中的 `roles[]` 数组的 `roleId` 和 `serverId` 字段。这是从绑定列表的深层结构中取的。

2. **`sk-game-role` header**：终末地签到必须携带此 header，格式为 `{gameId}_{roleId}_{serverId}`，其中 gameId 对终末地固定为 3。

3. **空 body 也要正确处理**：终末地签到端点不接受请求体，`body_str` 必须为空字符串 `""`，签名中对应位也为空。

### 调研方法

- 搜索 GitHub 上已有的终末地自动签到项目
- 参考了三个开源项目的实现：`sjtt2/endfield_auto_sign`、`xydesu/endfield-assistant`、`sglkc/endfield-auto-daily`
- 交叉验证后发现 API 端点、header 和签名参数完全一致，确认为正确方案
- 最后通过实际调用绑定 API 获取用户的 roleId 和 serverId 进行验证

### 经验总结

> **不要假设同一平台的子产品使用相同的 API。** 森空岛是鹰角的统一平台，但不同游戏（明日方舟、终末地）的签到接口、签名参数、请求格式完全不同。遇到未知的错误码（如 10001），应该先调研该产品是否有专用 API，而不是简单归类为"已签到"。

---

## 3. OAuth2 两步认证流程

森空岛的认证不是直接用 Token 调 API，而是需要两步换算：

```
鹰角通行证 Token
  → POST as.hypergryph.com/user/oauth2/v2/grant  → 授权码 (code)
  → POST zonai.skland.com/web/v1/user/auth/generate_cred_by_code  → cred + sign_token
```

- `cred`：后续所有 API 请求的身份凭证（放在 header 中）
- `sign_token`：签名密钥（用于计算请求签名）

### 为什么不直接存 cred？

直接存 cred 的问题：
- cred 有效期较短
- cred 过期后需要重新抓包获取

存 Token 的优势：
- Token 是鹰角通行证级别，有效期更长
- 通过 OAuth2 自动换算，更灵活
- Token 过期后重新获取更方便

---

## 4. Windows 定时任务的坑

### 路径问题

定时任务的执行目录默认是 `C:\Windows\System32`，不是脚本所在目录。因此所有文件路径必须使用**绝对路径**。

```python
# 错误 — 定时任务中找不到文件
with open("creds.txt", "r") as f: ...

# 正确 — 基于脚本位置构建绝对路径
script_dir = os.path.dirname(os.path.abspath(__file__))
creds_path = os.path.join(script_dir, "creds.txt")
```

### 静默运行

`python.exe` 运行时会弹出控制台窗口，用 `pythonw.exe` 则不会。但脚本中额外加了 `ctypes` 调用 `ShowWindow` 隐藏窗口作为双保险。

### ctypes 导入遗漏

脚本使用了 `ctypes.windll.user32.ShowWindow` 但最初忘记 `import ctypes`。因为被 try/except 包裹，错误被静默吞掉，导致排查困难。

> **教训**：try/except 吞掉异常很方便，但排查 bug 时很痛苦。开发阶段应该让异常暴露出来。

---

## 5. 代码架构设计

### SIGN_PROFILES 模式

不同游戏的签名参数差异很大，用一个字典 `SIGN_PROFILES` 存储参数，通过 `profile` 参数切换：

```python
SIGN_PROFILES = {
    "default":  {"platform": "1", "vName": "1.5.1", "dId": None},   # 明日方舟
    "endfield": {"platform": "3", "vName": "1.0.0", "dId": ""},     # 终末地
}
```

新增游戏只需在字典中添加一行，签名函数自动适配。

### 敏感信息分离

- `creds.txt`：存放 Token（敏感，不进版本控制）
- `config.json`：存放运行配置（无敏感信息，可进版本控制）

### 函数分层

```
工具层:  get_did(), log(), load_config(), load_tokens()
签名层:  compute_sign(), signed_request()
API 层:  get_cred_and_sign_token(), get_bindings()
业务层:  sign_attendance(), sign_endfield(), sign_checkin(), process_game()
入口层:  main()
```

每层只依赖下一层，职责清晰，便于维护和扩展。

---

## 6. 调试技巧

### 日志中的 Token 脱敏

```python
mask = token[:8] + "****" + token[-4:]
log(f"账号 ({mask})")
```

打印日志时只显示 Token 的前 8 位和后 4 位，避免泄露完整凭证。

### API 响应码含义

| code | 含义 |
|------|------|
| 0 | 成功 |
| 10001 | 今日已签到（非错误） |
| 其他 | 需要检查 `message` 字段 |

### 登岛检票是什么

"登岛检票"是森空岛平台自身的积分签到，与具体游戏的角色签到是独立的两套机制。每个游戏各有一次登岛检票机会（明日方舟 gameId=1，终末地 gameId=3）。

---

## 7. 扩展思路

如果鹰角未来推出新游戏需要签到，扩展步骤：

1. 抓包分析新游戏的签到 API 端点和参数
2. 在 `SIGN_PROFILES` 中添加新游戏的签名参数
3. 如果 API 格式与现有不同，新增对应的签到函数
4. 在 `process_game()` 中添加分支逻辑
5. 在 `CHECKIN_MAP` 中添加登岛检票的 gameId 映射
