# LAN File Download Center

局域网文件下载服务，跑在 Android 手机上。HTTP 浏览器直接打开就能下文件，附带 SFTP 服务可以当网络驱动器挂载。

> 当前版本 3.0。1.0 和 2.0 没有公开发布，直接从内部开发版跳到了 3.0。

## 能干啥

- HTTP 文件服务器，浏览器打开看列表、点下载，不用装客户端
- SFTP 服务器，可挂载为网络驱动器，支持只读 / 读写切换
- 下载前 MD5 二次校验，文件被改过或者损坏直接拦截，不下坏文件
- 每个文件有固定 8 位数字 ID，文件列表变动也不会下错东西
- 排队授权系统，有人请求下载得你手动同意，不想给就直接拒
- 文件黑名单，脚本自身、密钥这些敏感文件客户端看不到也下不了
- 桌面和手机浏览器自适应界面，支持按扩展名过滤、按时间 / 名称排序
- 全链路日志，谁连了、下了啥、啥时候断的都记着

## 运行环境

- Android 手机 + Termux（或其他能跑 Python 3 的环境）
- Python 3.8+
- HTTP 功能零额外依赖，纯标准库
- SFTP 功能需要 `paramiko >= 4.0.0`；没装也不影响 HTTP 跑，会自动降级跳过 SFTP

## Termux 环境准备

跑在手机上得先装 Termux，然后把编译工具和系统依赖装齐。paramiko 底层的 cryptography 要编译，缺库直接报错。

```bash
# 更新包管理器
pkg update && pkg upgrade

# 装 Python 和编译工具链
pkg install python python-pip clang make libffi openssl
```

如果 `pip install paramiko` 编译 cryptography 失败（常见报错 `Failed building wheel for cryptography`），有两条路：

**省事的办法**：直接装 Termux 预编译的 cryptography，再装 paramiko 时跳过依赖。

```bash
pkg install python-cryptography
pip install paramiko --no-deps
```

**硬编译的办法**：把编译工具装全，设置环境变量绕开某些报错。

```bash
pkg install clang make libffi openssl python-dev
export CFLAGS="-Wno-error=incompatible-function-pointer-types"
pip install paramiko
```

装完验证一下：

```bash
python -c "import paramiko; print(paramiko.__version__)"
```

能打印版本号就说明 SFTP 功能可用了。版本低于 4.0.0 的话脚本会拒绝启动 SFTP，得升级。

## 快速开始

```bash
# 装 SFTP 依赖（不装也能用 HTTP 部分）
pip install paramiko

# 启动
python3 auto_rp_download.py
```

启动后终端会打印访问地址，类似：

```
http://192.168.1.100:8192/
```

浏览器打开就能用。多网卡（WiFi + 热点）时会列出全部局域网 IP，选和客户端同网段的那个。

## 配置项

全部集中在脚本开头，改完保存重启生效：

| 配置 | 默认值 | 说明 |
|---|---|---|
| `SCAN_ROOT_DIR` | `/storage/emulated/0/Download/Browser/` | 扫描根目录，递归扫所有子目录 |
| `HTTP_PORT` | `8192` | HTTP 服务端口 |
| `SFTP_PORT` | `16384` | SFTP 服务端口 |
| `SFTP_USERNAME` | `admin` | SFTP 登录用户名 |
| `SFTP_PASSWORD` | `admin` | SFTP 登录密码 |
| `SFTP_READ_ONLY` | `True` | SFTP 只读模式；`True` 禁止上传 / 删除 / 改名 / 改权限 |
| `ALLOWED_EXTENSIONS` | （约 150 种后缀） | 允许扫描和下载的文件扩展名 |
| `ENABLE_HTML_INTERFACE` | `True` | 开浏览器界面；设为 `False` 进入终端命令授权模式 |
| `SFTP_FILE_BLACKLIST` | 脚本自身、密钥文件 | SFTP 黑名单，按文件名精确匹配，隐藏 + 拦截 |
| `SFTP_BLACKLIST_ENABLED` | `True` | 黑名单总开关；要传脚本到别的设备时临时改成 `False` |

## 几个容易踩的坑

- 默认 SFTP 是只读的，要上传文件得把 `SFTP_READ_ONLY` 改成 `False`
- `paramiko` 版本低于 4.0.0 会拒绝启动 SFTP，HTTP 部分不受影响，终端会打印安装指南
- 脚本自身和 `sftp_host_rsa_key` 默认在黑名单里，客户端看不到；需要把脚本传到别的设备时，把 `SFTP_BLACKLIST_ENABLED` 改成 `False`，传完改回来
- 手机别把 Termux 后台杀了，杀了服务就停。建议给 Termux 加电池优化白名单
- HTTP 传输是明文的，SFTP 才走 SSH 加密。公网环境下别直接暴露 HTTP 端口
- 下载授权排队默认最大并发 1，同一时间只能有一个客户端等待授权，其他人会排队

## 停止服务

终端里按 `Ctrl + C`，有信号处理，会优雅关闭 HTTP 和 SFTP，强制断开活跃连接，不会卡死。

## License

MIT
