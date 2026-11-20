# QQNT 聊天记录导出：完整复现文档（脱敏版）

> 一篇从零到可读 HTML / TXT 的完整实操记录。只保留技术内容，不含任何口令、账号与个人数据。

**适用范围**：Windows 10/11 x64 + 新版 QQ（QQNT / Electron 架构）+ 手机已把聊天记录**迁移**到该电脑的 QQNT 客户端。

> [!WARNING]
> 仅可用于**你本人拥有或已获授权处理**的数据。文档中的密钥、文件偏移（RVA）都与「本机 + 本账号 + 本 QQ 版本」绑定，换环境必须重新获取（见第 5 节）。

## 目录

- [0. 最终产物与验证基准](#0-最终产物与验证基准)
- [1. 原理总览](#1-原理总览)
- [2. 前置环境](#2-前置环境)
- [3. 定位数据](#3-定位数据三份备份迁移文件哪一份能用)
- [4. 环境准备](#4-环境准备)
- [5. 取密钥（核心步骤）](#5-取密钥核心步骤)
- [6. 解密数据库](#6-解密数据库)
- [7. 解析消息](#7-解析消息protobuf--文本)
- [8. 按会话导出并生成 HTML / TXT](#8-按会话导出并生成-html--txt)
- [9. 故障排查表](#9-故障排查表)
- [10. 复现清单](#10-复现清单checklist)
- [11. 安全与清理](#11-安全与清理)
- [附录 A](#附录-a本流程不使用的两条替代路线qce--napcat及其失败原因)

---

### 脱敏说明

本版本已移除全部敏感信息：
- 移除了加密口令（改为占位符，需自行抓取）
- 移除了 QQ 号、账号 UID、联系人备注名
- 移除了本机用户目录与自定义磁盘路径（改为 <占位符>）
保留的全部是技术内容（算法参数、函数定位方法、字段映射、脚本代码）。
换到其他机器 / 其他账号时，口令与函数地址都必须重新获取（见第 5 节）。

### 重要提醒

- **1.** 本文档已脱敏，不含真实密钥；你需要在目标机器上自行抓取（见第 5 节）。
- **2.** 密钥、文件偏移（RVA）都是"这一台机器 + 这一个账号 + 这一个 QQ 版本"的，
  换机器 / 换账号 / QQ 升级后必须重新获取（第 5 节有通用方法）。
- **3.** 只对本人拥有或获授权处理的数据进行此操作。

## 0. 最终产物与验证基准

产物：
<导出目录>\会话A_聊天记录.html     7.84 MB   42,242 条
<导出目录>\会话A_聊天记录.txt      1.59 MB
<导出目录>\会话B_聊天记录.html     9.53 MB   55,898 条
<导出目录>\会话B_聊天记录.txt      1.67 MB

中间产物（工作区 qqdecrypt\）：
nt_msg.db          128 MB   从 QQNT 目录复制出来的原始加密库
nt_msg_clear.db    128 MB   剥离 1024 字节头后的 SQLCipher 文件
nt_msg_plain.db     77 MB   解密后的明文 SQLite（可用任意工具打开）
nt_msg_export.db    42 MB   解析 Protobuf 后的结构化库（含 c2c_messages 表）

验证基准（复现成功时应得到相近数字）：
nt_msg_plain.db 中 sqlite_master 表数量：101
c2c_msg_table 行数：103,516
group_msg_table 行数：10,543
两个主要会话分别约 5.9 万条与 4.3 万条

## 1. 原理总览

QQNT（新版 QQ）把所有聊天记录存在一个 SQLCipher 加密的 SQLite 文件里：

<QQ数据目录>\<QQ号>\nt_qq\nt_db\nt_msg.db

它有两层保护：

(A) 1024 字节自定义头
文件头是 "SQLite header 3\0"（注意不是 "SQLite format 3"），
后面跟 "QQ_NT DB" 和一个哈希。这是腾讯加的壳，让标准工具认不出来。
必须先把这 1024 字节剥掉，剩下才是真正的 SQLCipher 文件。

(B) SQLCipher 4 加密
剥离后，文件开头 16 字节是随机 salt，之后按 4096 字节一页：
- 每页数据用 AES-256-CBC 加密
- 页尾保留 IV 和 HMAC 校验值
密钥 = PBKDF2 从"16 字节 ASCII 口令"和 salt 派生。

密钥不在任何文件里，只存在于 QQNT 进程内存中（QQ 打开数据库时传给
nt_sqlite3_key_v2 函数）。

所以流程是：
① 从 QQNT 进程内存抓 16 字节口令（第 5 节）
② 剥 1024 字节头（第 6 节）
③ 用 SQLCipher 4 参数解密并导出明文库（第 6 节）
④ 解析 40800 列的 Protobuf（第 7 节）
⑤ 按会话 UID 筛选并渲染（第 8 节）

## 2. 前置环境

### 实测环境

操作系统    ：Windows 10 家庭单语言版 22H2（内部版本 19045.7807）x64
权限        ：普通用户（非管理员）
QQNT 版本   ：9.9.37-54395（安装目录 <QQNT安装目录>）
QQ 数据目录 ：<QQ数据目录>（自定义位置，非默认 Documents）
wrapper.node：约 118 MB（118,186,536 字节）
Python      ：3.12.14
sqlcipher3  ：0.6.5（内置 SQLite 3.54.1，支持 SQLCipher 4 参数）
frida       ：17.27.1
protobuf    ：7.36.2（第三方解析工具 nt_msg_db_util 依赖）
其他        ：bbpb 1.4.2、python-dotenv 1.2.4

说明：以上版本仅为本次实测组合。其他 Windows 10/11 x64 + 相近 QQNT 版本
一般同样适用，关键只有一条——sqlcipher3 必须支持 SQLCipher 4。
本流程不需要 Node.js。

必要条件：
- QQNT 已安装且已登录该账号
- 聊天记录已通过手机 QQ「迁移聊天记录到电脑」写入 QQNT（不是"备份"）
- 有管理员/普通权限即可（Hook 同用户进程不需要管理员）

不需要（★ 重点，避免误解）：
- 不需要 QCE / QQChatExporter / NapCat / LiteLoaderQQNT 等任何"注入式"工具
- 不需要向 QQ 注入 DLL、不需要修改 QQ 任何文件
- 不需要匹配 QQ 版本兼容表（本方案与 QQ 版本无关）
- 不需要 QQ 密码
- 不需要联网上传任何数据
- 不需要 root 手机

本流程实际使用的工具清单（其余一律未使用）：
静态分析：任意 PE 工具 / Python 脚本（找字符串、算 RVA）
取密钥  ：frida（Python 包）
解密    ：sqlcipher3（Python 包，内置 SQLCipher 4）
解析    ：nt_msg_db_util（第三方开源仓库，纯 Python + protobuf）
导出    ：Python 自带 sqlite3 + 自写 HTML 生成脚本

## 3. 定位数据：三份「备份/迁移」文件，哪一份能用

用户通常会把三件事都做一遍，产生三种文件。它们完全不同：

【1】旧版 QQ 备份包（无用）
路径示例：<QQ数据目录>\<QQ号>\MsgBackup\
来源：手机 QQ「备份聊天记录到电脑」→ 旧版 QQ 9.7 的备份插件
（QQ\Plugin\Com.Tencent.MsgBackup\Bin\MsgBackup.dll）接收落盘
特征：哈希命名文件夹 + 4 个加密数据库（几百 MB 级）+ 哈希命名媒体子目录
能否读：不能。全加密，且腾讯官方说明"仅支持恢复至手机"

【2】新版 QQNT 的活数据库（★唯一可用★）
路径：<QQ数据目录>\<QQ号>\nt_qq\nt_db\nt_msg.db
来源：手机 QQ「迁移聊天记录到电脑」→ QQNT 直接写入自己的消息库
特征：单个 128 MB 左右的 .db 文件，同目录有 -wal / -shm 附属文件
能否读：能（本文档全流程处理的就是它）

【3】新版 QQNT 导出的 .bak 备份（无用）
路径示例：<任意位置>\<QQ号>的N个聊天记录.bak
来源：QQNT 客户端内「导出聊天记录」功能
特征：ZIP 结构，内含 db/（加密库）与 files/（加密媒体）
能否读：不能（需要同版本客户端导入；旧版 QQ 9.7 的"数据导入工具"
会报"消息记录导入失败，无法打开文件"）

判断口诀：
看目录里是不是有 nt_qq\nt_db\nt_msg.db —— 有，就用它；没有，先做"迁移"。

## 4. 环境准备

### 4.1 Python 依赖（必须用支持 SQLCipher 4 的包）

正确：pip install sqlcipher3
（PyPI 上有 Windows cp310~cp315 预编译轮子，本机装到的是 0.6.5，
内置 SQLite 3.54.1，支持 SQLCipher 4 参数）

错误（会失败，见第 9 节）：
pip install sqlcipher3-binary   ← 只有 manylinux 轮子，Windows 装不上
pip install pysqlcipher3-binary ← 只有 cp38 的 Windows 轮子，
但其内置 SQLCipher 是 3.x（SQLite 3.20），
不支持 SQLCipher 4 参数

### 4.2 取密钥依赖

pip install frida          （本机 17.27.1，Windows x64 有轮子）

### 4.3 Protobuf 解析工具

工具：https://github.com/QQBackup/nt_msg_db_util
（也可用 pip install protobuf bbpb python-dotenv 后直接跑它的 3.export.py）

下载：
https://codeload.github.com/QQBackup/nt_msg_db_util/zip/refs/heads/master
解压后主要文件：
1.decrypt.py   解密参考实现
2.slim.py      精简库
3.export.py    将明文库转成结构化库（本流程使用）
msgdb/         解析器代码（c2c / group / proto 定义）

### 4.4 可选：官方取密钥脚本（备用方法）

https://raw.githubusercontent.com/QQBackup/qq-win-db-key/master/scripts/windows/ntqq/windows_ntqq_get_key.ps1
用法（PowerShell）：
powershell -NoProfile -ExecutionPolicy Bypass -File windows_ntqq_get_key.ps1 -Verbose
说明：它静态分析 wrapper.node 找到 nt_sqlite3_key_v2 函数地址，然后用调试 API
启动 QQ 并下断点，输出 16 字节密钥。本机实测可用，但会把不可打印字符
显示成 '?'，需要配合方法 A 交叉验证。

## 5. 取密钥（核心步骤）

原理：QQNT 打开数据库时调用 nt_sqlite3_key_v2(db, zDbName, pKey, nKey)，
其中 pKey 指向 16 字节口令。我们在这个函数入口下断点，读寄存器。

Windows x64 调用约定：
rcx = db 句柄
rdx = 数据库名字符串指针（通常是 "main"）
r8  = pKey 指针     ← 我们要的
r9  = nKey 长度     ← 通常是 16

#### 5.1 先定位函数地址（必须自己重新算，不能照抄）

步骤：
1) 找到 wrapper.node：
<QQNT安装目录>\versions\<版本号>\resources\app\wrapper.node
（本机：<QQNT安装目录>\versions\9.9.37-54395\resources\app\wrapper.node
大小约 118 MB）

2) 在文件里搜索 ASCII 字符串：
nt_sqlite3_key_v2: db=%p zDb=%s
它位于 .rdata 段（本机文件偏移 0x490AD10，对应 RVA 0x490C710）。

3) 在 .text 段里找引用该地址的 LEA 指令（x64 是 48 8D 系列），
得到 LEA 指令 RVA（本机 0x22111F5）。

4) 用 PE 的异常目录（.pdata）把"函数起始地址"倒推出来：
在 .pdata 里二分查找包含该 RVA 的 (begin, end) 条目，
begin 就是函数起始 RVA（本机 0x22111E0）。

结论（仅本机有效）：
wrapper.node 内函数 RVA = 0x22111E0
运行时地址 = 模块基址 + 0x22111E0

换版本/换机器必须重做 2)~4)，不能直接用 0x22111E0。

#### 5.2 方法 A：Frida 动态 Hook（推荐，本机实际使用）

关键坑：wrapper.node 是 QQ 启动后才由 preload 加载的，
如果启动后立刻 attach 会"模块还没加载"，hook 不上。
正确做法是用模块加载观察器 Process.attachModuleObserver。

准备：QQ 先全部退出（任务管理器结束所有 QQ.exe）。

**脚本 hook_key.py（Python 侧）**

```python
import frida, time, json

EXE = r"<QQNT安装目录>\QQ.exe"
RVA = 0x22111E0          # ← 换成你算出来的函数 RVA
OUT = r"C:\temp\keys.json"

JS = """
const RVA = %d;
function hookModule(mod) {
  try {
    const addr = mod.base.add(RVA);
    send({info: 'hook ' + mod.name + ' base=' + mod.base});
    Interceptor.attach(addr, {
      onEnter: function (args) {
        try {
          const dbName = args[1].isNull() ? null : args[1].readUtf8String();
          const nKey = args[3].toInt32();
          let hex = null, ascii = null;
          if (!args[2].isNull() && nKey > 0 && nKey < 256) {
            const u8 = new Uint8Array(args[2].readByteArray(nKey));
            hex = Array.prototype.map.call(u8, b => ('0'+b.toString(16)).slice(-2)).join('');
            ascii = Array.prototype.map.call(u8, b => (b>=32&&b<127)?String.fromCharCode(b):'.').join('');
          }
          send({db: dbName, n: nKey, hex: hex, ascii: ascii});
        } catch (e) { send({err: String(e)}); }
      }
    });
  } catch (e) { send({err: 'hook fail: ' + e}); }
}
if (Process.attachModuleObserver) {
  Process.attachModuleObserver({
    onAdded: function (m) { if (m.name && m.name.indexOf('wrapper.node') >= 0) hookModule(m); }
  });
  send({info: 'observer installed'});
} else { send({err: 'frida too old'}); }
""" % RVA

results = []
def on_message(msg, data):
    if msg.get("type") == "send":
        p = msg["payload"]
        print(p, flush=True)
        if p.get("hex"): results.append(p); json.dump(results, open(OUT,"w"), ensure_ascii=False, indent=1)

pid = frida.spawn([EXE])
session = frida.attach(pid)
script = session.create_script(JS)
script.on("message", on_message)
script.load()
frida.resume(pid)

# 等待登录（若需要二维码登录，请在屏幕上完成登录）
deadline = time.time() + 240
while time.time() < deadline:
    time.sleep(2)
    if any((r.get("db") or "").lower().startswith("nt_msg") and r.get("hex") for r in results):
        break
```

运行：python hook_key.py
看到输出 CALL db=main n=16 key='....' 即成功。

说明：
- 会抓到两种 key：
n=20 → login.db 的固定密钥（对所有账号都一样，与消息无关，忽略）
n=16 → 我们要的消息库口令（会出现很多次，都一样）
- 本机抓到的 16 字节口令：<你的16字节口令>
（这是本机账号专属，换机器/换账号必然不同，请用上面的脚本重新抓）
- 必须在 QQ 正在运行、且已经打开数据库（即已登录）时才能抓到。

#### 5.3 方法 B：官方 PowerShell 调试器脚本（备用）

见 4.4。它用 Debug API 启动 QQ 并断点，输出字符串形式的密钥。
缺点：不可打印字符会显示成 '?'，需要配合方法 A 确认。

#### 5.4 方法 C：内存扫描（无 hook 时的兜底思路）

已知口令的一部分（比如从方法 B 得到一个残缺字符串）时：
- OpenProcess + ReadProcessMemory 枚举可读内存区
- 用已知片段做锚点搜索，取附近的 16 字节作为候选
- 逐个用第 6 节的方法试解，成功即命中
本机没有用到这一步，仅作为思路记录。

## 6. 解密数据库

### 6.1 复制数据库

建议先让 QQ 退出，再把以下文件一起复制（-wal 里可能有最后一点未合并数据）：

nt_msg.db
nt_msg.db-wal
nt_msg.db-shm

本机路径（复制到工作目录使用）：
<QQ数据目录>\<QQ号>\nt_qq\nt_db\nt_msg.db
→ C:\...\qqdecrypt\nt_msg.db   （128 MB）

### 6.2 剥离 1024 字节自定义头

**示例代码：**

```python
import os, shutil
src = r"C:\...\nt_msg.db"
dst = r"C:\...\nt_msg_clear.db"
with open(src,"rb") as f:
    head = f.read(1024)
    print(head[:32].hex())      # 应为 53514c697465206865616465722033 00
                                # 即 "SQLite header 3\0"
with open(src,"rb") as fin, open(dst,"wb") as fout:
    fin.seek(1024)
    shutil.copyfileobj(fin, fout)
```

剥离后，文件开头 16 字节就是 SQLCipher 的 salt（本机为 062dc2aa9b9558d8...）。

### 6.3 用正确参数解密并导出明文库

**本机实测可用的 PRAGMA 顺序（顺序很关键，不能乱）：**

```sql
PRAGMA cipher_page_size = 4096;
PRAGMA key = '<16字节口令>';          -- 口令里有单引号时，SQL 中写成 ''
PRAGMA kdf_iter = 4000;
PRAGMA cipher_hmac_algorithm = HMAC_SHA1;
PRAGMA cipher_kdf_algorithm = PBKDF2_HMAC_SHA512;
```

**完整脚本 decrypt_and_export.py**

```python
import os
from sqlcipher3 import dbapi2 as sq

ENC   = r"C:\...\nt_msg_clear.db"
PLAIN = r"C:\...\nt_msg_plain.db"
KEY   = "<你的16字节口令>"          # ← 换成你抓到的

if os.path.exists(PLAIN): os.remove(PLAIN)

con = sq.connect(ENC)
cur = con.cursor()
cur.execute("PRAGMA cipher_page_size = 4096;")
cur.execute("PRAGMA key = '%s';" % KEY.replace("'", "''"))
cur.execute("PRAGMA kdf_iter = 4000;")
cur.execute("PRAGMA cipher_hmac_algorithm = HMAC_SHA1;")
cur.execute("PRAGMA cipher_kdf_algorithm = PBKDF2_HMAC_SHA512;")

# 验证密钥是否正确：能读 sqlite_master 就说明对
cur.execute("SELECT count(*) FROM sqlite_master;")
print("tables:", cur.fetchone()[0])     # 本机应为 101

# 导出为无密码的明文库
cur.execute("ATTACH DATABASE '%s' AS plaintext KEY '';" % PLAIN.replace("\\","/"))
cur.execute("SELECT sqlcipher_export('plaintext');")
cur.execute("DETACH DATABASE plaintext;")
con.close()
print("done:", os.path.getsize(PLAIN)/1048576, "MB")   # 本机 77 MB
```

关键点：
- 必须用 sqlcipher3（SQLCipher 4），不能用 pysqlcipher3（3.x）
- key 里有单引号 '，SQL 字符串里要写成 '' 转义
- 参数不对时报错固定是："file is not a database"
- 导出用 sqlcipher_export() 最稳，不要逐表自己搬

## 7. 解析消息（Protobuf → 文本）

### 7.1 表结构与字段

明文库 nt_msg_plain.db 里主要两张消息表：

c2c_msg_table     私聊消息（本机 103,516 行）
group_msg_table   群聊消息（本机 10,543 行）

字段名是数字（腾讯风格），常用映射：

40001  消息 ID（整数主键）
40050  时间戳（Unix 秒）
40013  方向/状态（0=接收，2=发送等）
40020  发送者 UID
40033  发送者 QQ 号
40021  对方（会话）UID        ← 用它筛选某个会话
40030  对方（会话）QQ 号
40800  消息体（Protobuf 二进制）★ 内容都在这
40011  消息外层类型（2=普通文本、5=引用、6=名片、7=视频、9=无 content 的文本、
11=分享、19=通话、3=文件 等）

### 7.2 用现成解析器

解压 nt_msg_db_util 后，在仓库目录执行：

python 3.export.py --src <nt_msg_plain.db 绝对路径> --dst <nt_msg_export.db 绝对路径>

依赖：pip install protobuf bbpb python-dotenv

输出 nt_msg_export.db 里会多出：
c2c_messages      结构化私聊（含 text、content(JSON)、sender/peer、timestamp）
group_messages    结构化群聊
c2c_messages_fts  FTS5 全文索引

本机实测：c2c 103,516 行 + group 10,543 行，0 解析错误，耗时约 9 秒。

### 7.3 40800 里的 Protobuf 结构（自己写解析器时需要）

40800 = MsgBody，结构为 repeated MsgContent content = 1;
每个 MsgContent 里：
45101  文本内容（string）
45002  内容类型
其它类型（图片/视频/文件/语音/通话/名片/分享）解析成不同的 JSON 片段。

如果不想自己写，直接用 nt_msg_db_util 的 msgdb 包即可。

### 7.4 content 的常见 JSON 形态（导出后）

{"type":"text","text":"..."}                       纯文本
{"type":"image","filename":...,"local_path":...}    图片
{"type":"video","filename":...}                     视频
{"type":"call","desc":"通话时长 01:12:34"}          通话（只有时长，没有录音）
{"type":"file","filename":...}                      文件
{"type":"reply",...}                                引用（ref_summary 可能为空）
{"type":"mixed","segments":[...]}                   图文混排 / 多段
{"type":"sys","content":"[表情]"}                   系统/表情
{"type":"forward","meta":{...}}                     分享/合并转发
注意：很多行 content 为 NULL，但 text 列有值 —— 取文本时优先用 text。

## 8. 按会话导出并生成 HTML / TXT

### 8.1 找会话 UID

```sql
-- 按消息量排序，看有哪些会话
SELECT "40021" AS peer_uid, "40030" AS peer_qq, count(*) n
FROM c2c_msg_table GROUP BY peer_uid, peer_qq ORDER BY n DESC;

-- 看某会话的时间范围
SELECT min("40050"), max("40050"), count(*) FROM c2c_msg_table
WHERE "40021"='u_xxxxxxxx';

-- 把 UID 映射成备注名：解密 profile_info.db 后查询
--   profile_info.db 的加密方式与 nt_msg.db 相同（同样先剥 1024 字节头，
--   同样的 PRAGMA 参数和同一个密钥）
--   表 profile_info_v6：
--       1000 = UID, 1002 = QQ 号, 1001 = 昵称, 20002 = 备注
```

### 8.2 判定"谁说的"

sender_qq = 自己的 QQ 号（或 sender_uid = 自己的 UID）→ 我
否则 → 对方

### 8.3 生成 HTML（本机做法）

直接用 Python 从 nt_msg_export.db 读 c2c_messages：
- 筛选 peer_uid
- 渲染规则（建议）：
text 非空 → 直接显示
type=image → [图片]；video → [视频]；call → [通话] desc
file → [文件] 文件名；sticker → [表情]；contact → [名片]
reply → [回复] ref_summary；forward → [分享] 标题
sys → 居中灰字
- 输出：按日期分组 + 左右气泡 + 搜索框 + "只看文字"开关（纯 HTML+CSS+JS，可离线打开）

### 8.4 注意

- 原始条数 > 渲染条数：纯媒体/空包体行会被跳过（示例：原始 4.3 万条 → 渲染 4.2 万条）
- 媒体（图片/语音/视频）都在 nt_data 目录，可后续嵌图：
<QQ数据目录>\<QQ号>\nt_qq\nt_data\{Pic,Video,Ptt,Emoji,File}
- 语音条是 .amr 文件（可转码 + Whisper 转文字）；通话只有时长无音频

## 9. 故障排查表

说明：本节只列「本流程实际踩过的坑」。
关于 QCE / NapCat 那类「注入式路线」为什么不可行，见文末「附录 A」。

### 坑 1：pip install sqlcipher3-binary 失败

- **现象**：No matching distribution found
- **原因**：该包只有 manylinux 轮子，Windows 装不了
- **处理**：装 sqlcipher3（有 Windows 轮子）

### 坑 2：pysqlcipher3 装上但报 file is not a database

- **原因**：它内置 SQLCipher 3.x（SQLite 3.20），不支持 SQLCipher 4 的
HMAC_SHA1/PBKDF2_HMAC_SHA512 组合
- **处理**：换 sqlcipher3

### 坑 3：sqlite3mc 命令行解密失败

- **现象**：Parse error near ":..." / file is not a database
- **原因**：key 里含单引号，命令行拼接转义出错
- **处理**：改用 Python 驱动（sqlcipher3）传 PRAGMA，避免 shell 转义问题

### 坑 4：手工按页解密始终对不上

- **原因**：SQLCipher 的页布局（salt 前缀 / reserve 大小 / IV 位置）版本间不同，
自己实现极易错
- **处理**：直接用成熟库 sqlcipher3，别手撸

### 坑 5：Frida hook 报 wrapper.node not loaded

- **原因**：wrapper.node 是 QQ 启动后才加载的，attach 太早
- **处理**：用 Process.attachModuleObserver 等模块加载后再 hook；
或轮询 Process.findModuleByName 后 hook

### 坑 6：hook 到了但没有 nt_msg 的 key

- **原因**：QQ 未登录 / 数据库未打开
- **处理**：QQ 启动后需要完成登录，key 才会出现；20 字节那个是 login.db 的，
忽略它，等 16 字节的

### 坑 7：pip 安装时报临时目录无法删除

- **原因**：沙箱/权限限制
- **处理**：pip install --target <工作区目录> <包>，并把该目录加入 sys.path

### 坑 8：-wal 文件里还有数据

- **原因**：复制时 QQ 未干净退出
- **处理**：尽量退出 QQ 后复制；或接受最后几秒的数据缺失；
影响通常极小（本机 -wal 仅 0.45 MB）

### 坑 9：.bak 备份文件打不开

- **原因**：它是 QQNT 的加密备份容器，只能由对应版本客户端导入
- **处理**：放弃它，改用 nt_msg.db

## 10. 复现清单（Checklist）

- [ ] 1. 确认 QQNT 已安装并登录，手机已"迁移"记录到电脑
- [ ] 2. 找到 <QQ数据目录>\<QQ号>\nt_qq\nt_db\nt_msg.db（约 128 MB）
- [ ] 3. 退出 QQ，复制 nt_msg.db / -wal / -shm 到工作目录
- [ ] 4. 安装依赖：pip install sqlcipher3 frida protobuf bbpb
- [ ] 5. 下载 nt_msg_db_util 仓库
- [ ] 6. 静态分析 wrapper.node，定位 nt_sqlite3_key_v2 函数 RVA
- [ ] 7. 用 Frida（模块加载观察器）Hook，登录 QQ，抓 16 字节口令
- [ ] 8. 剥 1024 字节头 → nt_msg_clear.db
- [ ] 9. 按 4096/4000/HMAC_SHA1/PBKDF2_HMAC_SHA512 解密并 sqlcipher_export
- [ ] 10. 验证：sqlite_master 表数 ≈ 101，c2c_msg_table ≈ 103,516 行
- [ ] 11. 跑 3.export.py 生成结构化库
- [ ] 12. 查 peer_uid → 解 profile_info.db 映射备注名
- [ ] 13. 按会话生成 HTML / TXT
- [ ] 14. 清理明文库、解除敏感文件

## 11. 安全与清理

- nt_msg_plain.db / nt_msg_export.db 是全部聊天记录的明文，属于最敏感文件。
导出完成后建议：只保留最终 HTML/TXT，删除明文库；或加密存放。
- 抓到的 16 字节密钥只对当前机器当前账号有效，不要外传。
- 全程本地处理，不需要上传数据到任何服务器。
- 操作前建议先备份原始 nt_msg.db（只读复制即可）。

## 附录 A：本流程不使用的两条替代路线（QCE / NapCat）及其失败原因

说明：
本流程从始至终没有使用 QCE（QQChatExporter）、NapCat、LiteLoaderQQNT
或任何"注入式"工具，也没有调用它们的任何功能。
下面内容是常见但不可行的替代路线，列出来只为避免后来者重复踩坑；
只关心解密流程的读者可以完全跳过本附录。

### A-1：QCE / NapCat 一启动就把 QQ 杀掉

- **现象**：运行 QQChatExporter 后 QQ 进程消失；日志里不停刷
"NapCat WebUI 未响应（无端口监听）"
- **原因**：NapCat 启动流程需要独占 QQ，会执行 taskkill /im QQ.exe
- **影响**：只影响"想用 QCE 导出"的人，与本流程无关
- **处理**：不要使用注入路线；若机器上装过，先让它彻底退出并进行本流程

### A-2：QCE 报「文件已损坏，请重新安装 QQ」

- **现象**：QQ 启动后被完整性校验拦下，弹窗提示文件已损坏
- **原因**：QQ 版本超出 NapCat 支持的版本表
（本机 QQ 9.9.37-54395，而 NapCat 4.18.42 最高支持 9.9.37-54276），
注入修改后的启动清单无法通过校验
- **处理**：本方案绕开整个注入机制——不注入、不改 QQ 任何文件，
因此不受 QQ 版本兼容表限制

### A-3：NapCat 是什么，本流程为什么不需要它

NapCat 是"注入进 QQNT 进程、接管登录与消息收发"的框架，QCE 是它的插件。
它们能读到数据，是因为代码运行在 QQ 进程内部。

本流程换了一条等价但更稳的路：
只要 QQ 进程能打开数据库，密钥就一定存在于该进程内存中，
在 nt_sqlite3_key_v2 入口下 hook 即可拿到 —— 不需要修改 QQ 一个字节。
- **代价**：需要自己解 SQLCipher、自己解析 Protobuf（即本文档第 6、7 节的内容）。

