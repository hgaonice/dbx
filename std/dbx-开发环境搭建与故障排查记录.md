# DBX 开发环境搭建与故障排查记录

> 记录日期：2026-07-12
> 环境：Ubuntu 24.04 Server（无 GUI），root 用户
> 用途：汇总本次会话中涉及的所有操作命令，方便日后在新机器/新环境上复用，或排查同类问题时参考

---

## 一、项目启动流程

### 1.1 环境检查

```bash
node --version          # 要求 >= 22.13.0
pnpm --version           # 要求 10.27.0（package.json 的 packageManager 字段锁定）
rustc --version          # 要求 >= 1.77
cargo --version
which cmake gcc g++ pkg-config perl python3   # DuckDB bundled 编译需要这些
```

### 1.2 安装依赖

```bash
cd dbx
pnpm install --frozen-lockfile
```

### 1.3 启动开发服务器（无 GUI 服务器场景，跑 Web 模式）

```bash
# 终端 1：前端 Vite dev server（端口 5173）
make dev-web

# 终端 2：后端 Axum 服务（端口 4224）
make dev-backend
```

> 第一次跑 `make dev-backend` 会触发 DuckDB（以及 SQLite/SQLCipher/OpenSSL/AWS-LC）等 C/C++ 依赖从源码编译，耗时较长（本次实测约 14 分钟），详见第三、四节。

### 1.4 本机浏览器访问（SSH 端口转发）

服务器没有 GUI，需要在**本机**（有浏览器的机器）执行：

```bash
ssh -L 5173:localhost:5173 -L 4224:localhost:4224 用户名@服务器地址
```

转发建立后，本机浏览器打开 `http://localhost:5173` 即可访问 dbx 界面。

### 1.5 可选：加快后端热重载

```bash
cargo install cargo-watch
```

装好后 `make dev-backend` 会自动检测并启用热重载。

### 1.6 可选：起测试数据库联调

```bash
docker run -d --name test-mysql -e MYSQL_ROOT_PASSWORD=test -p 3306:3306 mysql:8
```

之后在 dbx 里用 `localhost:3306` 连接即可（前后端和数据库都在同一台机器上）。

### 1.7 只想跳过 DuckDB 快速检查/开发（不需要 DuckDB 功能时）

```bash
make cargo-check-fast    # cargo check --no-default-features
make cargo-test-fast     # cargo test --no-default-features
cargo run -p dbx-web --no-default-features --features mq-admin,sqlite-sqlcipher
```

> ⚠️ 注意：`cargo-check-fast`/`cargo-test-fast` 是在**整个 workspace**（不加 `-p`）上跑 `--no-default-features`，会连带检查 `src-tauri`（Tauri 桌面壳层），而 `src-tauri` 即使不用默认 features，编译时仍然需要 `libwebkit2gtk-4.1-dev libgtk-3-dev libappindicator3-dev librsvg2-dev patchelf libssl-dev libsecret-1-dev` 这些 GTK/WebKit 系统库（Linux 桌面构建依赖，见 `README.zh-CN.md` 和 `.github/workflows/ci.yml`）。纯服务器场景如果没装这些库，建议用 `-p` 显式只检查目标 crate，例如：
>
> ```bash
> cargo check -p dbx-web
> cargo check -p dbx-core
> ```

### 1.8 `cargo check` 和 `cargo run` 的区别（想调试一定要用 run，不是 check）

| 命令 | 作用 | 能不能拿来调试 |
|---|---|---|
| `cargo check -p dbx-web` | 只做类型检查/编译校验，**不产出可执行文件** | 不能，跑不起服务 |
| `cargo build -p dbx-web` | 编译并生成可执行文件，但不运行 | 不能，还要手动运行 |
| `cargo run -p dbx-web` | 编译 + **实际运行起来** | ✅ 这才是调试要用的，`make dev-backend` 本质就是包了一层这个命令 |

验证是否已经产出可执行文件：
```bash
ls target/debug/dbx-web 2>&1   # cargo check 不会生成这个文件，cargo run/build 会
```

### 1.9 桌面模式 `make`/`make dev` 会自动带起前端，但跟 `dbx-web` 无关

默认目标 `make`（等价 `make dev`）执行的是 `pnpm dev:tauri` → `tauri dev`，它会：
1. 自动执行 `tauri.conf.json` 里的 `beforeDevCommand`（即 `pnpm dev`，启动 Vite 前端，端口 1420）
2. 等前端就绪后，编译并启动 Tauri 桌面 Rust 程序，打开原生窗口加载前端

**关键点**：桌面模式下前端通过 Tauri IPC（`invoke`）直接调用 `src-tauri` 内嵌的 Rust 函数，**完全不需要 `dbx-web`**（`dbx-web` 只有 Web/Docker 模式才用得到）。

> ⚠️ 无 GUI 的服务器上跑不了 `make`：一是 `src-tauri` 编译需要 `libwebkit2gtk-4.1-dev` 等系统库（验证命令：`pkg-config --exists webkit2gtk-4.1`，没装会报 `glib-2.0`/`gobject-2.0` not found），二是 `printenv DISPLAY` 是空的，没有窗口系统可以渲染原生窗口。这台服务器上开发调试应该继续用 `dev-web` + `dev-backend`，`make` 留给本机 Windows/macOS 或带桌面环境的 Linux。

### 1.10 局域网访问前端失败：Vite 默认只绑定 `127.0.0.1`

**现象**：`make dev-web` 起来后，SSH 端口转发访问 `localhost:5173`没问题，但局域网内其他机器直接访问 `http://<服务器IP>:5173/` 打不开。

**排查命令**：
```bash
ss -tlnp | grep -E ":5173|:4224"
```
如果看到：
```
LISTEN  0.0.0.0:4224   ← dbx-web 后端，监听所有网卡，局域网能访问
LISTEN  127.0.0.1:5173 ← Vite 前端，只监听回环地址，局域网访问不到
```
就是这个问题。

**根因**：`apps/desktop/vite.config.ts` 里 `server.host` 取决于 `TAURI_DEV_HOST` 环境变量，`make dev-web`/`pnpm dev:web` 没设置它，所以 Vite 默认只绑定 `127.0.0.1`。

**❌ 错误的修复方式（不生效，是个坑）**：
```bash
pnpm dev:web -- --host 0.0.0.0
```
`pnpm run <script> -- <参数>` 这种写法，pnpm 会把 `--` 字符本身也原样拼进最终命令，变成：
```
vite --config apps/desktop/vite.config.ts --port 5173 --mode web -- --host 0.0.0.0
```
中间多出来的裸 `--` 被 Vite 的命令行解析器（cac）当成了「停止解析选项，后面全部当普通位置参数」的信号，所以 `--host 0.0.0.0` 根本没被识别成 `--host` 参数，等于没生效，端口仍然绑在 `127.0.0.1`。

**✅ 正确修复方式：跳过 pnpm 脚本包装，直接调用 vite**
```bash
pnpm exec vite --config apps/desktop/vite.config.ts --port 5173 --mode web --host 0.0.0.0
```

重新确认监听地址（应该变成 `0.0.0.0:5173`）：
```bash
ss -tlnp | grep 5173
```

Vite 自己的启动日志也会直接打印出局域网可访问的地址，方便确认：
```
VITE v8.0.16   web   ready in 360 ms

➜  Local:   http://localhost:5173/
➜  Network: http://192.168.137.250:5173/
```

> ℹ️ 已经确认不是防火墙问题：`ufw status` 是 `inactive`，`iptables -L -n` 的 `INPUT` 链默认策略是 `ACCEPT`。局域网访问不到，99% 是 Vite/Node 服务本身绑定在 `127.0.0.1` 导致的，遇到同类问题先用 `ss -tlnp` 确认监听地址，而不是急着排查防火墙。

### 1.11 长期跑：把前后端放到后台，日志写文件，方便随时查看/重启

开发调试时不想占着两个终端，可以用 `nohup` + `setsid` 放到后台跑，日志重定向到文件：

```bash
# 后端
cd dbx
setsid nohup make dev-backend > /tmp/dbx-backend.log 2>&1 < /dev/null &

# 前端（记得带 --host，否则局域网访问不到，见 1.10）
setsid nohup pnpm exec vite --config apps/desktop/vite.config.ts --port 5173 --mode web --host 0.0.0.0 > /tmp/dbx-frontend.log 2>&1 < /dev/null &
```

查看日志 / 确认启动状态：
```bash
tail -f /tmp/dbx-backend.log     # 实时看后端日志，Ctrl+C 退出查看不影响后台运行
tail -f /tmp/dbx-frontend.log    # 实时看前端日志
ss -tlnp | grep -E ":5173|:4224"  # 确认端口监听状态
```

停止后台进程：
```bash
ps aux | grep -E "vite|dbx-web" | grep -v grep    # 找到 PID
kill <PID>
```

---

## 二、Git 相关操作

### 2.1 HTTPS 认证问题（密码认证已被 GitHub 取消）

现象：
```
remote: Invalid username or token. Password authentication is not supported for Git operations.
fatal: Authentication failed for 'https://github.com/xxx/xxx.git/'
```

原因：GitHub 从 2021 年 8 月起，HTTPS 方式的 git 操作不再支持账号密码认证，必须用 **Personal Access Token (PAT)** 或 **SSH**。

排查命令：
```bash
git --no-pager remote -v
git config --get credential.helper
git config --global --get credential.helper
```

解决方式（三选一）：
1. 生成 PAT（网页操作，无 CLI 命令）：https://github.com/settings/tokens/new → 勾选 `repo` 权限 → 生成后在 push 弹出的 Password 栏粘贴 token
2. 切换成 SSH：
   ```bash
   git remote set-url origin git@github.com:用户名/仓库.git
   ```
3. 装 GitHub CLI，用 `gh auth login` 自动配置（见 2.4 节）

> ⚠️ **重要提醒**：Token 属于敏感凭据，不要粘贴在聊天记录、代码仓库、日志里。一旦泄露（哪怕只是发给 AI 助手），应立即在 https://github.com/settings/tokens 里 Delete 掉，重新生成。

### 2.2 本地仓库对象损坏（合并中断导致 0 字节坏对象）

现象：
```
fatal: bad object <sha>
fatal: the remote end hung up unexpectedly
send-pack: unexpected disconnect while reading sideband packet
```

根因：`git merge` 执行过程中进程被中断（磁盘满/容器重启/被杀），导致合并提交对象和某个树对象在磁盘上写成了 **0 字节文件**。

**诊断命令：**
```bash
git rev-parse --is-shallow-repository
git cat-file -t <可疑的 sha>              # 报错 "object corrupt or missing" 说明对象坏了
git --no-pager fsck --full --no-dangling   # 全面检查仓库完整性
git --no-pager fsck --full | sort -u       # 全量检查，去重看错误类型
cat .git/HEAD                              # 看 HEAD 指向哪个 ref
cat .git/refs/heads/<分支名>                # 看分支指针指向的 sha
cat .git/logs/HEAD                          # 看 reflog，找损坏发生前的最后一个健康提交
```

**修复步骤（分支指针回退到合并前的健康提交）：**
```bash
# 1. 确认回退目标提交是健康的
git cat-file -t <合并前的健康 sha>

# 2. 把分支指针改回健康提交（不影响工作区文件）
git update-ref refs/heads/<分支名> <合并前的健康 sha>

# 3. 重建索引（不改动工作区，只是让索引和 HEAD 对齐，此时能看到 merge 引入的改动变成 "modified"/"untracked"）
git reset --mixed HEAD
git --no-pager status --short --branch
```

**如果索引/工作区里有 merge 遗留的孤儿文件（未提交内容想整体回滚）：**
```bash
git reset --hard HEAD              # 丢弃已跟踪文件的所有改动
git reset --hard <目标 sha>         # 如需进一步对齐到某个健康提交（比如远程分支的 tip）
git clean -fd -n                   # 预览会删除哪些未跟踪文件/目录（-n 是 dry-run）
git clean -fd                      # 真正删除所有未跟踪文件/目录（不可恢复，谨慎使用）
```

**遇到 `index.lock` 残留（上一次 git 命令被中断留下的锁文件）：**
```bash
ps aux | grep -i git              # 先确认没有其他 git 进程真的在跑
rm -f .git/index.lock             # 确认没有进程占用后再删除锁文件
```

**清理历史 reflog 里指向已损坏对象的记录（可选，纯粹让 fsck 输出干净）：**
```bash
git reflog expire --expire=now --all
git gc --prune=now
```

**重新执行合并并推送：**
```bash
git merge main
git push origin <分支名>
```

### 2.3 用 Token 临时推送（不落盘持久化，用完立即清除）

```bash
# 临时把 token 写进远程地址，仅用于这一次 push
git remote set-url origin https://<token>@github.com/用户名/仓库.git
git push origin <分支名>
# 推送完立刻恢复成不带 token 的地址，避免 token 明文留在 .git/config 里
git remote set-url origin https://github.com/用户名/仓库.git
```

> ⚠️ 凡是经手过明文 token 的操作，事后都要去 GitHub 网页端 Delete 掉该 token，重新生成。

### 2.4 安装并使用 GitHub CLI（`gh`），告别手动复制 Token

**Ubuntu/Debian 安装命令：**
```bash
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg -o /usr/share/keyrings/githubcli-archive-keyring.gpg
chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" > /etc/apt/sources.list.d/github-cli.list
apt update
apt install gh -y
gh --version
```

**登录认证（交互式，需要手动操作，不能脚本化）：**
```bash
gh auth login
```
跟随提示选择：
1. `GitHub.com`
2. 协议选 `HTTPS`
3. `Authenticate Git with your GitHub credentials?` 选 **Yes**（自动帮 git 配置凭据）
4. 认证方式选 `Login with a web browser`，按提示在浏览器里输入一次性验证码完成授权

**确认登录状态：**
```bash
gh auth status
```

登录成功后，`git push`/`git pull` 会自动使用 `gh` 管理的凭据，不再需要手动粘贴 token。

---

## 三、DuckDB 编译说明

### 3.1 用在哪里

| 用途 | 相关文件 |
|---|---|
| 作为一种可连接的数据库类型（DuckDB 文件 / 内存数据库） | `crates/dbx-core/src/db/duckdb_driver.rs` |
| DuckDB 连接的进程隔离（子进程运行，崩溃不影响主进程） | `crates/dbx-core/src/db/duckdb_worker_process.rs`、`duckdb_worker_protocol.rs`、`duckdb_worker_runtime.rs` |
| 拖拽 Parquet/CSV/JSON 文件即时预览（用 DuckDB 直接查询文件） | README 功能特性「文件预览」 |

### 3.2 为什么必须编译进去，不能像 MySQL 那样单纯连接

- MySQL / PostgreSQL / Redis / MongoDB 等是**独立服务器进程**，dbx 只需要实现网络协议客户端（轻量），不需要把数据库本身编译进来。
- DuckDB（和 SQLite 一样）是**嵌入式数据库**，没有独立服务进程，本质是一个 C++ 库，必须把它的执行引擎、存储引擎源码编译并静态链接进 dbx 的可执行文件才能使用。
- Rust 的 `duckdb` crate 提供 `bundled` 模式（把 DuckDB C++ 源码整个编译进来，用户开箱即用，无需自装 DuckDB）。dbx 的 `Cargo.toml` 里只暴露了这一种模式（`duckdb-bundled` feature），没有暴露"链接系统已装 DuckDB"的选项，所以**无法通过预装系统 DuckDB 来跳过编译**。

### 3.3 项目里其他同类"内嵌编译"依赖

| 依赖 | Feature | 用途 |
|---|---|---|
| `rusqlite`（SQLite） | `bundled` | SQLite 也是嵌入式数据库，同样的道理必须编译进来 |
| `rusqlite/bundled-sqlcipher-vendored-openssl` | `sqlite-sqlcipher` | SQLite 加密扩展 SQLCipher + 内嵌 OpenSSL（给数据库文件加密用，跟 HTTPS 无关） |
| `rustls` 的 `aws-lc-rs` | 默认开启 | TLS 加密后端（给 MySQL/PostgreSQL/Redis 等的加密连接用），基于 AWS-LC，需要 cmake/nasm 编译 C 代码 |
| `zstd-sys`、`freetype-sys` 等 | 间接依赖 | 分别来自压缩（`zip`/`calamine`）和字体渲染（`font-kit`），体量较小 |

判断规律：Cargo.toml 里出现 `bundled`、`vendored`，或者依赖名带 `-sys` 后缀，基本都是"源码编译进最终二进制"的信号。

### 3.4 相关 Cargo.toml 配置（`crates/dbx-core/Cargo.toml`）

```toml
[features]
default = ["duckdb-bundled", "mq-admin", "sqlite-sqlcipher"]
duckdb-bundled = ["duckdb/bundled"]
sqlite-sqlcipher = ["rusqlite/bundled-sqlcipher-vendored-openssl"]

[dependencies]
duckdb = { version = "1.3.2", optional = true }
rusqlite = { version = "0.32", features = ["bundled", "load_extension", "backup", "functions", "column_decltype"] }
rustls = { version = "0.23", features = ["aws-lc-rs"] }
```

---

## 四、sccache 缓存编译：安装、配置与使用

### 4.1 为什么要装

`cargo clean` 会删掉 `target/`，之后重新编译 DuckDB/SQLite/OpenSSL/AWS-LC 这些 C/C++ 依赖又要等很久。`sccache` 按**源码内容哈希**缓存编译产物，缓存目录在 `target/` 之外（默认 `~/.cache/sccache`），就算 `cargo clean` 也不会清空，重新编译能直接命中缓存，一劳永逸。

> 不建议用 `cargo install sccache`（会现场编译，耗时很长），直接下载官方预编译二进制更快。

### 4.2 安装（下载预编译二进制，以 Linux x86_64 为例）

```bash
# 查最新版本号：https://github.com/mozilla/sccache/releases/latest
cd /tmp
curl -fsSL -o sccache.tar.gz https://github.com/mozilla/sccache/releases/download/v0.16.0/sccache-v0.16.0-x86_64-unknown-linux-musl.tar.gz
tar xzf sccache.tar.gz
cp sccache-v0.16.0-x86_64-unknown-linux-musl/sccache /usr/local/bin/sccache
chmod +x /usr/local/bin/sccache
sccache --version
```

### 4.3 配置 Cargo 使用 sccache

写入 `~/.cargo/config.toml`（如果已有内容，先备份再追加，不要整个覆盖）：

```toml
[build]
rustc-wrapper = "/usr/local/bin/sccache"

[env]
CC = "sccache cc"
CXX = "sccache c++"
```

- `rustc-wrapper`：让 Cargo 编译 Rust 代码时经过 sccache
- `CC`/`CXX`：让 `cc` crate（`libduckdb-sys`/`libsqlite3-sys`/`openssl-sys`/`aws-lc-sys` 等 build script 用的 C/C++ 编译工具链）也经过 sccache 缓存

### 4.4 ⚠️ 重要坑：第一次加 `rustc-wrapper` 会触发全量重新编译

只要是**第一次**给已经编译过的项目加上 `rustc-wrapper`，Cargo 会认为所有编译单元的"指纹"变了（因为编译命令本身多了个 wrapper），从而触发**整个依赖树**（不只是 DuckDB，是所有 Rust crate + 所有 C/C++ 依赖）的一次性全量重编译。

本次实测：加了 sccache 配置后，`cargo check -p dbx-web` 从零开始全量编译，耗时 **12 分 49 秒**（因为部分内容已经被前一次编译顺带缓存，实际可能因机器而异，预期 20-40 分钟量级）。**这是一次性代价**，编译完成后 sccache 缓存会填满，以后 `cargo clean` 之后重建能大幅命中缓存。

建议做法：**在项目刚 clone 下来、还没编译过任何东西之前**就配置好 sccache，可以避免这次"二次全量编译"的额外开销。

### 4.5 常用命令

```bash
sccache --show-stats     # 查看缓存命中率、命中/未命中次数等统计
sccache --stop-server    # 停止 sccache 后台服务进程
```

`--show-stats` 关键字段说明：
- `Cache hits` / `Cache misses`：命中/未命中次数，命中越多说明缓存越有效
- `Cache hits rate`：命中率
- `Cache location`：缓存存放路径（默认 `~/.cache/sccache`）
- `Max cache size`：缓存上限（默认 10 GiB，够存 DuckDB/SQLite/OpenSSL/AWS-LC 的编译产物）

本次会话验证结果（第一次全量重编译后）：
```
Compile requests                   4688
Cache hits                         2785
Cache misses                       1881
Cache hits rate                   59.69 %
```

---

## 五、后台长时间编译的排查经验

### 5.1 “看起来卡住”但其实在正常编译

如果执行 `cargo check ... | tail -n N` 感觉毫无输出、卡住不动，大概率是：
1. **DuckDB/SQLite/OpenSSL 等 C/C++ 源码正在编译**（体量大，10~20 分钟量级很正常）
2. `tail`（非 `-f` 模式）会等到整个命令输出流关闭才打印内容，中间过程的 "Compiling xxx" 进度全被缓冲，看起来像没反应

**排查是否真的在跑（而不是真卡死）：**
```bash
ps aux | grep -E "cargo|rustc|cc1|c\+\+" | grep -v grep
```
能看到 `cargo check`、`libduckdb-sys` 的 build-script、`c++ ... .cpp` 之类的进程，说明是正常编译，耐心等即可。

### 5.2 长时间编译建议放后台跑，避免阻塞终端/工具超时

```bash
cd dbx
nohup cargo check -p dbx-web > /tmp/cargo-check.log 2>&1 &
```

查看实时进度（不影响后台编译）：
```bash
tail -f /tmp/cargo-check.log
```

确认是否编译完成：
```bash
tail -n 5 /tmp/cargo-check.log     # 看到 "Finished `dev` profile ..." 说明编译成功完成
ps aux | grep "cargo check" | grep -v grep    # 没有输出说明进程已经结束
```
