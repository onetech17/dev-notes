# 企业代理环境配置与故障排查

> 本文档整理了两次关于公司代理环境的诊断与解决方案，涵盖 `git push --mirror` 失败的根因分析、代理配置项正确性验证、以及可落地的修复方案。
>
> 注：文中代理域名、内网域名与仓库地址均已脱敏为占位符，替换为真实值即可复用。

---

## 目录

- [背景与网络环境](#背景与网络环境)
- [第一部分：`git push --mirror` 报错排查](#第一部分git-push---mirror-报错排查)
  - [问题现象](#问题现象)
  - [诊断过程](#诊断过程)
  - [根本原因](#根本原因)
  - [解决方案](#解决方案)
- [第二部分：代理配置项正确性验证](#第二部分代理配置项正确性验证)
  - [待验证的配置](#待验证的配置)
  - [逐项验证结果](#逐项验证结果)
  - [关键发现](#关键发现)
- [推荐的完整修复方案](#推荐的完整修复方案)
- [附录：速查命令](#附录速查命令)

---

## 背景与网络环境

工作机所在网络环境受限：

- **DNS 解析**：GitHub 域名可解析（指向真实公网 IP，如 `20.205.243.166`）
- **直接出网**：HTTP/HTTPS/SSH 直连 GitHub 均失败
- **公司代理**：仅允许通过 `proxy.corp.example.com` 代理出网
  - 80 端口：HTTP 代理（仅支持 HTTP/FTP 流量，不支持 HTTPS CONNECT 隧道）
  - 1080 端口：SOCKS5 代理（支持任意 TCP 流量，含 HTTPS）
- **内网不走代理**：通过 `no_proxy` 配置了公司内网域名与私网地址段

---

## 第一部分：`git push --mirror` 报错排查

### 问题现象

执行 `git push --mirror origin` 时报错：

```text
fatal: 无法访问 'https://github.com/<你的用户名>/dev-notes.git/'：Proxy CONNECT aborted
```

### 诊断过程

#### 1. 查看现有代理配置

```bash
git config --global --get http.proxy    # 空
git config --global --get https.proxy   # 空
echo "HTTP_PROXY=$HTTP_PROXY"
echo "HTTPS_PROXY=$HTTPS_PROXY"
echo "http_proxy=$http_proxy"
echo "https_proxy=$https_proxy"
```

输出：

```text
HTTP_PROXY=
HTTPS_PROXY=
http_proxy=http://proxy.corp.example.com:80
https_proxy=http://proxy.corp.example.com:80
```

#### 2. 直接连通性测试

| 测试项 | 命令 | 结果 |
|---|---|---|
| 直连 GitHub HTTPS | `curl https://github.com` | `000`（超时/失败） |
| 走 HTTP 代理访问 HTTPS | `curl -x http://proxy.corp.example.com:80 https://github.com` | `000`（CONNECT 被中止） |

#### 3. 代理服务本身的可用性

```bash
curl -v --max-time 10 -x "http://proxy.corp.example.com:80" http://github.com
```

返回 `301 Moved Permanently` → 代理服务器本身正常。

#### 4. HTTPS CONNECT 隧道测试

```bash
curl -v --max-time 15 -x "http://proxy.corp.example.com:80" \
    https://github.com/<你的用户名>/dev-notes.git/info/refs?service=git-upload-pack
```

关键日志：

```text
> CONNECT github.com:443 HTTP/1.1
> Host: github.com:443
* Proxy CONNECT aborted
curl: (56) Proxy CONNECT aborted
```

#### 5. 探测其它可用代理路径

```bash
# SSH 22 端口
nc -zv -w 5 github.com 22
# → Connection timed out（SSH 被屏蔽）

# SOCKS5 代理 (1080 端口)
curl -s -o /dev/null -w "%{http_code}\n" --max-time 10 \
    -x "socks5://proxy.corp.example.com:1080" https://github.com
# → 200（成功）
```

### 根本原因

```text
报错信息：fatal: 无法访问 'https://github.com/<你的用户名>/dev-notes.git/'：Proxy CONNECT aborted

根本原因：
  1. 网络只能通过 SOCKS5 代理 (proxy.corp.example.com:1080) 访问 GitHub
  2. HTTP 代理 (proxy.corp.example.com:80) 不支持 HTTPS CONNECT 隧道
  3. SSH 22 端口被防火墙屏蔽
  4. Git 默认走环境变量里的 HTTP 代理 → CONNECT 被中止 → 失败
```

### 解决方案

#### 方法 1：单次命令使用 SOCKS5 代理（最简单）

```bash
git -c http.proxy=socks5://proxy.corp.example.com:1080 \
    -c https.proxy=socks5://proxy.corp.example.com:1080 \
    push --mirror origin
```

#### 方法 2：Git 全局代理（永久生效）

```bash
git config --global http.proxy socks5://proxy.corp.example.com:1080
git config --global https.proxy socks5://proxy.corp.example.com:1080
# 之后直接 git push --mirror origin
```

#### 方法 3：环境变量（针对单次 shell 会话）

```bash
export ALL_PROXY=socks5://proxy.corp.example.com:1080
git push --mirror origin
```

#### 验证方案有效

```bash
GIT_CURL_VERBOSE=1 git -c http.proxy=socks5://proxy.corp.example.com:1080 \
    -c https.proxy=socks5://proxy.corp.example.com:1080 \
    ls-remote https://github.com/<你的用户名>/dev-notes.git
```

输出包含：

```text
* SOCKS5 communication to github.com:443
* SOCKS5 request granted.
* SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
* SSL certificate verify ok.
< HTTP/1.1 200 OK
```

---

## 第二部分：代理配置项正确性验证

### 待验证的配置

| 配置项 | 值 |
|---|---|
| HTTP 代理 | `http://proxy.corp.example.com` 端口 `80` |
| HTTPS 代理 | `http://proxy.corp.example.com` 端口 `80` |
| FTP 代理 | `http://proxy.corp.example.com` 端口 `80` |
| SOCKS 代理 | `socks5://proxy.corp.example.com` 端口 `1080` |

### 逐项验证结果

#### HTTP 代理（80 端口）

| 测试场景 | 状态 |
|---|---|
| 代理 HTTP 流量（`http://httpbin.org/ip`） | ✅ 200 |
| 代理 HTTPS 流量（`https://github.com`，需 CONNECT 隧道） | ❌ `Proxy CONNECT aborted` |

#### HTTPS 代理（80 端口）

> "HTTPS 代理"≠"代理服务器本身用 HTTPS"。这里指代理 HTTPS **流量**，协议头仍是 `http://`。

| 测试场景 | 状态 |
|---|---|
| 代理 HTTPS 流量（CONNECT 隧道） | ❌ 不支持（与 HTTP 代理共享 80 端口，同一限制） |

#### FTP 代理（80 端口）

| 测试场景 | 状态 |
|---|---|
| 代理 FTP 流量（`ftp://ftp.gnu.org/`） | ✅ 200 |

#### SOCKS5 代理（1080 端口）

| 测试场景 | 状态 |
|---|---|
| 代理 HTTP 流量 | ✅ 200 |
| 代理 HTTPS 流量 | ✅ 200 |

### 验证汇总表

```text
代理                    协议          端口    HTTP    HTTPS   FTP
─────────────────────────────────────────────────────────────
proxy.corp.example.com     http://        80      ✅      ❌     ✅
proxy.corp.example.com     socks5://    1080      ✅      ✅     ✅
```

### 关键发现

#### 问题 1：「HTTPS 代理」配置的歧义

```text
你写的：HTTPS 代理  http://proxy.corp.example.com  端口 80
实际测：HTTP 代理（端口 80）不能处理 HTTPS CONNECT 隧道
```

关于"HTTPS 代理"这个名字：

- ❌ 不是「代理服务器本身用 HTTPS 协议」（那样应该写 `https://...`）
- ✅ 真正的意思是「代理 HTTPS **流量**」（即让 HTTPS 请求走这个代理）

格式本身是对的，但实际验证发现这个 80 端口代理**不支持 HTTPS CONNECT 隧道方法**。

#### 问题 2：环境变量里没有 SOCKS 代理

```text
当前环境：
  http_proxy    = http://proxy.corp.example.com:80    ✓ 存在
  https_proxy   = http://proxy.corp.example.com:80    ✓ 存在（但对 HTTPS 无效）
  SOCKS_PROXY   = (未设置)                         ❌ 缺失
```

系统里有 SOCKS5 代理（端口 1080），但**没有导出为环境变量**，所以 git、curl 默认都走 HTTP 代理。

#### 问题 3：GNOME 系统代理设置未启用

```bash
gsettings get org.gnome.system.proxy.mode      # → 'none'
gsettings get org.gnome.system.proxy.http.host # → ''
gsettings get org.gnome.system.proxy.http.port # → 8080（默认占位）
```

GUI 网络设置中的代理配置为空。

---

## 推荐的完整修复方案

### 方案 A：环境变量修正（立即生效，仅当前终端）

```bash
export http_proxy="http://proxy.corp.example.com:80"
export https_proxy="socks5://proxy.corp.example.com:1080"   # ← 关键：用 SOCKS5
export ftp_proxy="http://proxy.corp.example.com:80"
export SOCKS_PROXY="socks5://proxy.corp.example.com:1080"
export ALL_PROXY="socks5://proxy.corp.example.com:1080"
export no_proxy="localhost,127.0.0.0/8,*.corp.example.com,.corp.example.com,corp.example.com,10.*.*.*"
```

### 方案 B：永久写入 ~/.bashrc（推荐）

```bash
cat >> ~/.bashrc << 'EOF'

# ===== 公司代理配置 =====
export http_proxy="http://proxy.corp.example.com:80"
export https_proxy="socks5://proxy.corp.example.com:1080"
export ftp_proxy="http://proxy.corp.example.com:80"
export SOCKS_PROXY="socks5://proxy.corp.example.com:1080"
export ALL_PROXY="socks5://proxy.corp.example.com:1080"
export no_proxy="localhost,127.0.0.0/8,*.corp.example.com,.corp.example.com,corp.example.com,10.*.*.*,git.internal.example.com"
EOF

source ~/.bashrc
```

### 方案 C：Git 专用配置

```bash
git config --global http.proxy socks5://proxy.corp.example.com:1080
git config --global https.proxy socks5://proxy.corp.example.com:1080
```

### 方案 D：清理旧的 HTTP 代理配置（如果之前设错）

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
git config --global http.proxy socks5://proxy.corp.example.com:1080
git config --global https.proxy socks5://proxy.corp.example.com:1080
```

---

## 附录：速查命令

### 代理连通性测试

```bash
# HTTP 代理测试 HTTP 流量
curl -s -o /dev/null -w "%{http_code}\n" --max-time 10 \
    -x "http://proxy.corp.example.com:80" http://httpbin.org/ip

# HTTP 代理测试 HTTPS 流量（会失败）
curl -s -o /dev/null -w "%{http_code}\n" --max-time 10 \
    -x "http://proxy.corp.example.com:80" https://github.com

# SOCKS5 代理测试 HTTPS 流量（会成功）
curl -s -o /dev/null -w "%{http_code}\n" --max-time 10 \
    -x "socks5://proxy.corp.example.com:1080" https://github.com
```

### 查看当前代理配置

```bash
# Git 配置
git config --global --get http.proxy
git config --global --get https.proxy

# 环境变量
echo "$http_proxy $https_proxy $SOCKS_PROXY $no_proxy"

# GNOME 系统代理
gsettings list-recursively org.gnome.system.proxy
```

### Git 操作通过 SOCKS5 代理

```bash
# 单次命令
git -c http.proxy=socks5://proxy.corp.example.com:1080 \
    -c https.proxy=socks5://proxy.corp.example.com:1080 \
    push --mirror origin

# 全局配置后
git push --mirror origin
```

### 验证 HTTPS CONNECT 隧道是否被代理支持

```bash
curl -v --max-time 15 -x "http://proxy.corp.example.com:80" \
    https://github.com 2>&1 | grep -E "CONNECT|aborted|HTTP/1.1"
```

看到 `Proxy CONNECT aborted` 即表示该 HTTP 代理不支持 HTTPS 隧道。

---

## 一句话总结

> 在公司代理环境下访问 GitHub 等 HTTPS 站点，**HTTP 代理（80 端口）无法代理 HTTPS 流量**，必须使用 **SOCKS5 代理（1080 端口）**；配置时将 `http_proxy` 设为 `http://proxy.corp.example.com:80`，将 `https_proxy` 和 `ALL_PROXY` 设为 `socks5://proxy.corp.example.com:1080`。