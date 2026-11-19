---
name: dingtalk-extraction
description: 提取钉钉 PC 本地聊天库消息（解密 db+wal、按会话/群导出文本、卡片、图片路径）。触发词：钉钉提取消息、解密钉钉、钉钉本地数据库、班级群消息导出、dingwave、dingtalk.db。
---

# 钉钉本地聊天库提取

从钉钉 PC 客户端本地加密库中**只读**提取任意会话/群的消息。已实测环境：钉钉 PC（2026-10，库版本 V2，Windows 11）。

## 红线（先读再动手）

- **只读副本**：全程操作复制出来的文件，绝不直接打开/修改原 db（原库被客户端占用且写入会损坏证据）。
- **禁发令**：对任何钉钉群/会话只读，任何情况下不发送消息（含测试消息）。
- **skill 内不放任何凭证**；端点 key 一律运行时从 CC Switch 库或环境读取。
- 图片可能含他人（含未成年人）信息：OCR 优先本地离线引擎；上云识别需用户明确授权。

## 核心链路（四步）

参考实现：`D:\Claude\项目\01 参谋长\08-家族\念念\钉钉班级群日报\_构建脚本\dingtalk_db.py`（纯 Python，已实测）。

1. **定位**：`%APPDATA%\DingTalk\{UID}_v2\DBFiles\dingtalk.db`，UID = 目录名数字段（如 `4735600_v2` → UID=4735600）。
2. **复制副本**：db 与 `dingtalk.db-wal` 都要复制（新消息在 WAL 里！只复制主库最多读到上次 checkpoint，可能落后数天）。先复制主库再复制 WAL；WAL 复制时被截断的最坏结果只是旧快照，不会出错数据。
3. **解密**：key = `md5(UID).hexdigest()[:16]`（16 字节 ASCII），整库按 **4096 字节页 AES-128-ECB** 解密；不足整页的尾部原样拷贝。依赖 `pycryptodome`。1 秒可解 26MB，无需 dingwave 二进制。
4. **WAL 重放**（关键，缺一不可）：
   - WAL 头 32 字节（明文的），salt = 字节 16:24；帧结构 = `[24B 帧头 + 4096B 页]` 循环。
   - 逐帧校验 salt，帧页同样 AES-ECB 解密；按 pgno 写入主库镜像（`(pgno-1)*4096 : pgno*4096`）。
   - **只应用到"最后提交帧"**（帧头 bytes4:8 非零 = 提交帧，其值为提交后库总页数 dbsize）；超出最后提交帧的未提交帧丢弃。
   - **先按 dbsize 扩容主库镜像再写入**（bytearray 越界切片赋值会错位追加，直接导致 database disk image is malformed）。
   - 重放后 sqlite 正常打开；可用 99 提交点完整性校验做验证。

## 查询要点

- 会话清单 `tbconversation`（cid/title/type）；按 title 精确匹配找群（如"一年级1班"）。
- 消息在 128 张分表 `tbmsg_000~127`，按 cid 全表扫描聚合；`createdAt` 是毫秒时间戳。
- 发送者昵称：`tbuser_group_nick`（cid+uid→groupNick，老师常带"数学老师"等后缀；泛称"家长"时回退 `tbuser_profile_v2` 的 nick/realName）。
- contentType 口径：1=文本（content JSON.text）/ 2=图片 / 2950=互动通知卡片 / 1202=群合并卡片（作业提交）/ 202、3100=视频 / 1201、1200=卡片。卡片正文在 `attachments[0].extension`（已是 dict，也可能再嵌一层 JSON 字符串），可取 `interactiveCardLastMessage`、`groupMergeMarkdown`、`title`。
- 图片消息 content JSON 有 `blurredPath`（小模糊预览，≤120px，OCR 无意义）和 `url`（down.dingtalk.com 需登录态，匿名 404）；原图不自动落本地。要拿原图需登录态下载（见下）。

## 已知坑位

- dingwave（GitHub p1g3/dingwave）可用来快速验证：`-d 副本 -k UID -o 输出`，但解密完会**常驻起 Web 服务 8080**（不是卡死），subprocess 用要自行 taskkill。
- 手工重放 WAL 必须**先解密主库基底**再重放（混放报 file is not a database）。
- senderId 是数字，join 昵称表要转字符串。
- 钉钉大版本升级可能换加密方案（社区已有 V3=PBKDF2 的库出现）；失效时先拿副本跑通再改算法，不要在原文件上试。

## 扩展

- 在线表格/AI 表格取数：见全局 MEMORY-dev-recipes「钉钉AI表格（notable）OpenAPI取数配方」。
- 图片原图下载：dingwave 带 `-token` 参数（需账号 token，从客户端存储提取，属凭证操作，需用户单独授权）。
- 日报管道完整实战（抓取→OCR→LLM 日报→飞书推送→计划任务）：见 `D:\Claude\项目\01 参谋长\08-家族\念念\钉钉班级群日报\` 项目。
