# WorkBuddy 每日自动签到（GitHub Actions 版）

把 WorkBuddy「Buddy 加油站」每日签到 + 派猫猫旅行做成 **GitHub Actions 定时任务**。

**核心卖点：不开机也能签。** 任务跑在 GitHub 的服务器上，你自己的电脑关机、客户端没开都不影响，每天北京时间 09:00 自动完成签到领积分。

底层复用 [totorosir 的 WorkBuddy 签到助手](https://www.workbuddy.cn) 技能脚本（通道 C：长效令牌自续期，纯 Python 标准库、零依赖），只把"谁来定时跑"换成了 GitHub Actions。

---

## 原理

| 项 | 说明 |
|---|---|
| 运行位置 | GitHub 服务器（`ubuntu-latest`），与你本机无关 |
| 触发方式 | `schedule` cron：每天 UTC 01:00 = 北京时间 09:00 |
| 鉴权方式 | 通道 C 长效令牌（refresh token），一次性配置后长期自续 |
| 是否需要客户端 | **不需要**，客户端关着也能签 |
| 是否需要开机 | **不需要** |

> 为什么不用本机定时任务 / WorkBuddy 自带自动化？因为它们都要求你电脑开机、客户端在线。本方案就是要"不开机"，所以选 GitHub Actions。

---

## 部署步骤（一次性）

### 1. 获取你的长效令牌（refresh token）

在你**平时登录了 WorkBuddy 客户端的 Windows 电脑**上操作（仅需一次）：

```bash
# 进入本仓库的 scripts 目录后执行，明文打印你的长效令牌
python scripts/rt_auth.py --export-rt
```

复制终端输出的那一长串令牌（形似 `xxxxx.yyyyy.zzzzz` 的 base64 串，等同账号钥匙，勿外泄）。

> 如果你之前已经在本地 `--export-rt --save` 过，也可以直接打开
> `C:\Users\<你>\.workbuddy\checkin-rt.json`，复制里面的 `rt` 字段值。

### 2. 把令牌存进 GitHub 仓库 Secret

1. 打开你的仓库 → **Settings → Secrets and variables → Actions → New repository secret**
2. Name 填：`WORKBUDDY_RT`
3. Secret 填：第 1 步复制的那串令牌
4. 保存

令牌**只存在 Secret 里**，不会进入代码、不会出现在日志（脚本与 Action 均已脱敏）。

### 3. 启用工作流

- 首次推送后，进入仓库 **Actions** 标签页，找到 `WorkBuddy 每日自动签到`，点 **Enable workflow**。
- 点一次 **Run workflow** 手动跑一遍验证（看日志里 `status` 是否为 `ok`）。

之后每天 09:00（北京时间）自动运行，无需任何操作。

---

## 验证结果

每次运行的输出是 JSON，关键字段：

- `"status": "ok"` + `"action": "clicked"` → 今天签到成功，+100 积分
- `"action": "skip_already_signed"` → 今天已签过（含手动签），安全跳过
- `"status": "error"` + 提到"长效令牌刷新失败" → 令牌失效，回到第 1 步重新导出并更新 Secret

派猫猫旅行结果在 `"travel"` 字段里。

---

## 可选：签到失败/成功时推送通知

默认 `--no-notify`，只看 Action 日志。想收到钉钉/飞书/微信等推送：

1. 参考 `templates/notify_config.json.example` 写好配置。
2. 把整个 `notify_config.json` 内容存成另一个仓库 Secret（如 `NOTIFY_CONFIG`），
   并改 workflow 里最后一步为：
   ```yaml
   - name: 执行签到（带推送）
     run: |
       echo '${{ secrets.NOTIFY_CONFIG }}' > ~/.workbuddy/scripts/notify_config.json
       python3 scripts/rt_auth.py
   ```
3. 详见原技能 `references/user-guide.md` 的推送章节。

---

## 注意事项 / 坑

- **cron 可能延迟**：GitHub 在高负载时会把定时任务延迟几分钟到几十分钟，属正常；极端情况可手动 Run workflow 补签（签到幂等，重复跑不会多扣）。
- **60 天 inactive 自动停用**：GitHub 会在仓库约 60 天无活动后自动停用 scheduled workflow。若发现停了，进 Actions 页面重新 Enable，或随便推一个空提交即可恢复。
- **令牌失效**：refresh token 由官方签发、长期有效；若某天报"刷新失败"，按第 1 步重新导出并更新 `WORKBUDDY_RT` Secret 即可。
- **免费额度**：私有仓库的 GitHub Actions 有免费额度，每日一次远用不完。

---

## 文件结构

```
.github/workflows/daily-checkin.yml   # 定时任务定义（每天 09:00 北京时间）
scripts/                              # 签到助手脚本（通道 A/B/C + 推送模块）
templates/                            # 推送配置模板 + 网页授权码工具
README.md
```

脚本版权归原技能作者 totorosir（公众号「龙猫科技说」）所有，本仓库仅做 GitHub Actions 托管封装。
