# 一、目标场景说明

公司环境要求：

* Ubuntu 终端**默认只能走内网**
* 不能全局设置系统代理
* 需要在命令行中访问外网（GitHub / HuggingFace / pip / conda）
* 使用 SOCKS5 代理：

```text
192.168.5.93:15732
```

---

# 二、核心问题总结（你今天遇到的坑）

## ❌ 问题1：proxychains4 单独运行报错

```bash
proxychains4
```

报：

```
can't load process....: No such file or directory
```

### 原因

proxychains 必须后接命令：

```bash
proxychains4 wget https://xxx
```

---

## ❌ 问题2：wget 报 “不支持 socks5h”

报错：

```
不支持的协议类型 “socks5h”
```

### 原因（关键）

环境变量污染：

```bash
http_proxy=socks5h://192.168.x.xx:xxxx
https_proxy=socks5h://192.168.x.xx:xxxx
ALL_PROXY=socks5h://192.168.x.xx:xxxx
```

wget 不支持 `socks5h://`，会自己解析失败。

---

## ❌ 问题3：proxychains + 环境变量冲突

即使 proxychains 正常，也会被：

```bash
http_proxy / https_proxy
```

干扰。

---

## ❌ 问题4：proxychains 仍显示 socks5h 报错

原因：

> proxychains 已经接管，但 wget 仍然读取环境变量

---

## ❌ 问题5：DNS 显示 224.0.0.1

原因：

* DNS 没完全走代理
* proxy_dns 需要开启

---

# 三、正确解决方案（完整流程）

---

# STEP 1：安装 proxychains4

```bash
sudo apt update
sudo apt install proxychains4 -y
```

---

# STEP 2：修改配置文件

```bash
sudo nano /etc/proxychains4.conf
```

---

## 修改3个关键点

---

### 1）开启 DNS 代理（必须）

```ini
proxy_dns
```

---

### 2）使用 dynamic_chain（推荐）

```ini
#strict_chain
dynamic_chain
```

---

### 3）配置 SOCKS5 代理

在最下面：

```ini
[ProxyList]
socks5 192.168.x.xx:xxxx
```

---

# STEP 3：清理环境变量（关键）

每次使用前必须保证干净环境：

```bash
unset http_proxy
unset https_proxy
unset HTTP_PROXY
unset HTTPS_PROXY
unset ALL_PROXY
unset all_proxy
```

---

# STEP 4：验证代理是否正常

```bash
proxychains4 curl https://api.ipify.org
```

或：

```bash
proxychains4 curl https://www.google.com
```

看到返回 HTML 或 IP 即成功。

---

# STEP 5：正确使用方式（核心）

## ❌ 错误用法

```bash
proxychains4 wget https://xxx
```

（如果环境变量污染，会失败）

---

## ✅ 正确用法（推荐）

### 方法1：用 alias（最推荐）

---

## 写入 ~/.bashrc

```bash
nano ~/.bashrc
```

追加：

```bash
alias pwget='env -u http_proxy -u https_proxy -u ALL_PROXY proxychains4 wget'
alias pgit='env -u http_proxy -u https_proxy -u ALL_PROXY proxychains4 git'
alias ppip='env -u http_proxy -u https_proxy -u ALL_PROXY proxychains4 pip'
alias pcurl='env -u http_proxy -u https_proxy -u ALL_PROXY proxychains4 curl'
alias pconda='env -u http_proxy -u https_proxy -u ALL_PROXY proxychains4 conda'
```

---

## 生效：

```bash
source ~/.bashrc
```

---

# 四、标准使用方法（给同事直接用）

---

## 1）下载文件（HuggingFace / 模型）

```bash
pwget https://huggingface.co/gpt2/resolve/main/config.json
```

---

## 2）Git 拉代码

```bash
pgit clone https://github.com/ultralytics/ultralytics.git
```

---

## 3）pip 安装

```bash
ppip install torch torchvision
ppip install ultralytics
```

---

## 4）conda 使用

```bash
pconda create -n yolo python=3.10
pconda install numpy
```

---

## 5）curl 请求

```bash
pcurl https://api.ipify.org
```

---

# 五、验证是否成功（必须做）

## 测试 HuggingFace：

```bash
pwget https://huggingface.co/gpt2/resolve/main/config.json
```

成功输出：

```
HTTP 200 OK
saved file
```

---

## 测试 GitHub：

```bash
pgit clone https://github.com/ultralytics/ultralytics.git
```

---

## 测试 pip：

```bash
ppip install numpy
```

---

# 六、系统结构总结（非常重要，给理解用）

你当前网络结构是：

```text
Ubuntu
  ↓
proxychains4（劫持 socket）
  ↓
SOCKS5代理（192.168.x.xx:xxxx）
  ↓
外网（GitHub / HF / pip）
```

同时：

```text
普通命令 → 直接走公司内网
加 pwget → 才走代理
```

---

# 七、关键经验总结（避免别人踩坑）

### 1️⃣ 绝对不要全局设置 socks5h

```bash
http_proxy=socks5h://...
```

❌ 会导致 wget / pip 报错

---

### 2️⃣ proxychains + 环境变量必须隔离

必须：

```bash
env -u http_proxy ...
```

---

### 3️⃣ 永远用 alias 封装

避免手动输入错误

---

### 4️⃣ DNS 必须开 proxy_dns

否则：

```
解析到 224.0.0.1（异常）
```

---


