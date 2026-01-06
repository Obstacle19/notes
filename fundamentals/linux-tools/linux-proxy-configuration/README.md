# Linux 服务器代理配置方法

[TOC]

## 1. 下载客户端

- 在用户目录下创建 `clash` 文件夹：

```shell
# 创建 clash 文件夹
[~]$ mkdir ~/clash
[~]$ cd ~/clash
```

- 在以下页面中下载合适的 **Clash-Premium** 版本压缩包：
  👉 [Release Clash-Premium · DustinWin/proxy-tools · GitHub](https://github.com/DustinWin/proxy-tools/releases/tag/Clash-Premium)
- 将压缩包复制到 `~/clash` 目录下
- 解压缩、重命名并修改权限：

```shell
# 解压缩
[~/clash]$ tar -zxvf clashpremium-release-linux-amd64-v3.tar.gz

# 重命名为 clash
[~/clash]$ mv CrashCore clash

# 确保权限正确
[~/clash]$ chmod +x clash
```

## 2. 下载 Clash 配置文件

- 下载 Clash 配置文件 `config.yaml`：

```shell
[~/clash]$ wget -O config.yaml "https://cb3wv.no-mad-world.club/link/1kEiPhW9xxxxxxxx?clash=3"
```

- 确认 `config.yaml` 中 `allow-lan: false`，否则会导致流量异常消耗

## 3. 运行 Clash

- 在 `clash` 目录下运行二进制文件：

```shell
[~/clash]$ ./clash -d .
```

- 若提示缺失 `Country.mmdb`，可从镜像源下载：

```bash
[~/clash]$ wget -4 -O Country.mmdb https://gitlab.com/ineo6/geoip/raw/master/Country.mmdb
```

- 最终目录结构应如下所示：

```text
.
|-- Country.mmdb
|-- cache.db
|-- clash
|-- clashpremium-release-linux-amd64-v3.tar.gz
`-- config.yaml
```

- 运行成功后的界面示例如下：

![fig1](figs/clash.jpg)

## 4. 修改代理环境变量

### 4.1 临时设置代理

- Clash 默认端口一般为 `7890`，可在 `config.yaml` 中通过 `port: 7890` 查看。

```shell
export HTTP_PROXY=http://127.0.0.1:7890
export HTTPS_PROXY=http://127.0.0.1:7890
```

- 同时设置大小写版本以兼容不同程序：

```shell
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
export HTTP_PROXY=http://127.0.0.1:7890
export HTTPS_PROXY=http://127.0.0.1:7890
export NO_PROXY=localhost,127.0.0.1,::1

# 让 sudo 继承代理变量（核心）
alias sudo='sudo -E'  # -E 表示继承当前用户环境变量
```

### 4.2 永久生效配置

- 编辑用户环境变量文件：

```shell
vim ~/.bashrc
```

- 在文件末尾添加：

```shell
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
export HTTP_PROXY=http://127.0.0.1:7890
export HTTPS_PROXY=http://127.0.0.1:7890
export NO_PROXY=localhost,127.0.0.1,::1
alias sudo='sudo -E'
```

- 使配置生效：

```shell
source ~/.bashrc
```

### 4.3 验证代理是否生效

- 向 `www.google.com` 发送 `GET` 请求进行验证：

```shell
[~]$ curl https://www.google.com
```