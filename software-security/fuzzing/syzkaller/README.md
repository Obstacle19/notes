# Syzkaller 的配置和使用

[TOC]

> **测试环境**：
>
> Host：Ubuntu 22.04 x86_64
>
> Kernel：Linux 6.2
>
> syzkaller: master (commit e14dbeb9)
>
> QEMU：qemu-system-x86_64

## 1. 编译 syzkaller

### 1.1 下载并配置 go 语言编译器

1. 下载并解压缩 `go` 语言编译器

```shell
# 下载 go 语言编译器压缩包
[~]$ wget https://dl.google.com/go/go1.23.6.linux-amd64.tar.gz

# 解压缩，完成后会在 ~ 目录下看到 go 文件夹
[~]$ tar -xf go1.23.6.linux-amd64.tar.gz

# 修改文件夹名为 go1.23.6
[~]$ mv go ~/go1.23.6
```

2. 配置 `go` 环境变量

```shell
# 临时配置 (不推荐)
[~]$ export GOROOT=$HOME/go1.23.6 # 注意 `pwd` 需要替换，如替换为 $HOME
[~]$ export PATH=$GOROOT/bin:$PATH
[~]$ export GOPATH=$HOME/gowork
[~]$ export GOMODCACHE=$GOPATH/pkg/mod
[~]$ export PATH=$PATH:$GOPATH/bin

# 永久配置 (推荐)
[~]$ echo 'export GOROOT=$HOME/go1.23.6' >> ~/.bashrc
[~]$ echo 'export PATH=$GOROOT/bin:$PATH' >> ~/.bashrc
[~]$ echo 'export GOPATH=$HOME/gowork' >> ~/.bashrc
[~]$ echo 'export GOMODCACHE=$GOPATH/pkg/mod' >> ~/.bashrc
[~]$ echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.bashrc
[~]$ source ~/.bashrc # 生效配置
```

3. 验证 `go` 环境配置

```shell
# 验证 go 版本
[~]$ go version
```

### 1.2 下载编译 syzkaller

1. 下载和编译 `syzkaller`

```shell
[~]$ git clone https://github.com/google/syzkaller
[~]$ cd syzkaller
[~/syzkaller]$ make
```

2. 在 `syzkaller/vm/qemu/qemu.go` 文件中添加 `"net.ifnames=0"` 字段，通过内核启动参数 `CmdLine`  添加 `net.ifnames=0`，强制使用传统网卡命名（`eth0`），确保网络服务正常启动并分配 IP

```go
// syzkaller/vm/qemu/qemu.go
var archConfigs = map[string]*archConfig{
	"linux/amd64": {
		Qemu:     "qemu-system-x86_64",
		QemuArgs: "-enable-kvm -cpu host,migratable=off",
		// e1000e fails on recent Debian distros with:
		// Initialization of device e1000e failed: failed to find romfile "efi-e1000e.rom
		// But other arches don't use e1000e, e.g. arm64 uses virtio by default.
		NetDev: "e1000",
		RngDev: "virtio-rng-pci",
		CmdLine: []string{
			"root=/dev/sda",
			"console=ttyS0",
			"earlyprintk=serial net.ifnames=0",
		},
	},
	...
```

## 2. 编译 linux 内核

1. 以 `linux 6.2` 内核为例，先将源码下载到本地

```shell
[~]$ git clone --branch v6.2 git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

2. 生成默认配置文件


```shell
[~]$ cd linux

# 生成默认配置
[~/linux]$ make defconfig

# 叠加 KVM 虚拟机适配配置
[~/linux]$ make kvm_guest.config
```

3. 更改 `linux` 目录下的 `.config` 文件，在最后面添加如下代码

```ini
# Coverage collection.
CONFIG_KCOV=y

# Debug info for symbolization.
CONFIG_DEBUG_INFO_DWARF4=y

# Memory bug detector
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y

# Required for Debian Stretch and later
CONFIG_CONFIGFS_FS=y
CONFIG_SECURITYFS=y
```

4. 重新生成配置文件并编译


```shell
# 合并手动追加的配置到 .config，自动补全依赖项
[~/linux]$ make olddefconfig

# 根据可用处理器核心数并行编译 linux 内核
[~/linux]$ make -j`nproc`
```

5. 等待一段时间，编译完成后，就能在目录下看到`vmlinux` (kernel binary) 和 `bzImage` (packed kernel image)


```shell
[~/linux]$ ls vmlinux
[~/linux]$ ls arch/x86/boot/bzImage
```

## 3. 安装 qemu 虚拟机

1. 安装 `debootstrap`


```shell
[~]$ sudo apt install debootstrap
```

2. 使用 `create-image.sh` 构建 `syzkaller` `qemu` 镜像


```shell
[~]$ mkdir -p image
[~]$ cd image

# 下载 syzkaller 用来制作 qemu 虚拟机磁盘镜像的脚本 create-image.sh
# 若下载失败，则访问 https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh 手动下载
[~/image]$ wget https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh -O create-image.sh
[~/image]$ chmod +x create-image.sh

# 下载 Debian 的 bookworm 版本
[~/image]$ ./create-image.sh --distribution bookworm

# 修改密钥文件的权限
[~/image]$ chmod 600 bookworm.id_rsa
```

3. 安装 `qemu`


```shell
[~]$ sudo apt install qemu-system-x86
```

4. 在 `.bashrc` 中定义相关变量，便于后续使用


```bash
# ~/.bashrc
export KERNEL=~/linux          # 编译好的内核目录（含bzImage）
export IMAGE=~/image           # bookworm.img 所在目录
```

```shell
[~]$ source ~/.bashrc # 生效配置
```

5. 确保内核能够`sshd`启动
   - 在虚拟机中，使用 `poweroff` 或 `shutdown -h now` 正常关机（`exit` 仅退出 `shell`，不一定关闭虚拟机）
   - 在宿主机中，按 `Ctrl + A` 再按 `X` 关闭 `qemu`


```shell
qemu-system-x86_64 \
	-m 2G \
	-smp 2 \
	-kernel $KERNEL/arch/x86/boot/bzImage \
	-append "console=ttyS0 root=/dev/sda earlyprintk=serial net.ifnames=0" \
	-drive file=$IMAGE/bookworm.img,format=raw \
	-net user,host=10.0.2.10,hostfwd=tcp:127.0.0.1:10021-:22 \
	-net nic,model=e1000 \
	-enable-kvm \
	-nographic \
	-pidfile vm.pid \
	2>&1 | tee vm.log

# syz-manager 实际使用的 QEMU 启动参数（debug 时可在日志中看到）
qemu-system-x86_64 -m 4096 -smp 2 -chardev socket,id=SOCKSYZ,server=on,wait=off,host=localhost,port=4523 -mon chardev=SOCKSYZ,mode=control  -display none -serial stdio -no-reboot -name VM-0  -device virtio-rng-pci -enable-kvm -cpu host,migratable=off  -net user,host=10.0.2.10,hostfwd=tcp:127.0.0.1:10021-:22 -net nic,model=e1000 -hda /home/ss/image/bookworm.img -snapshot -kernel /home/ss/linux/arch/x86/boot/bzImage -append "console=ttyS0 root=/dev/sda earlyprintk=serial "
```

6. 之后可以通过 `ssh` 在另一个终端中连接到 `qemu` 实例。打开另一个终端


```shell
ssh -i $IMAGE/bookworm.id_rsa -p 10021 -o "StrictHostKeyChecking no" root@localhost
```

## 4. 启动 syzkaller - qemu

1. 创建工作目录 `workdir`


```shell
[~/syzkaller]$ mkdir workdir
[~/syzkaller]$ vim my.cfg
```

2. 在 `my.cfg` 中输入以下内容：


```ini
{
	"target": "linux/amd64",
	"http": "127.0.0.1:56741",
	"workdir": "/home/ss/syzkaller/workdir",
	"kernel_obj": "/home/ss/linux",
	"image": "/home/ss/image/bookworm.img",
	"sshkey": "/home/ss/image/bookworm.id_rsa",
	"syzkaller": "/home/ss/syzkaller",
	"procs": 8,  
	"type": "qemu",  
	"vm": {
		"count": 4,  
		"kernel": "/home/ss/linux/arch/x86/boot/bzImage",
		"cpu": 2,
		"mem": 4096
	}
}
```

3. 把用户加入到 `kvm` 组

```shell
sudo usermod -aG kvm ss
# 注销重登录
groups
```

4. 运行 `syzkaller` 管理器：


```shell
[~/syzkaller]$ ./bin/syz-manager -config=my.cfg

# debug 模式下使用
[~/syzkaller]$ ./bin/syz-manager -config=my.cfg -debug
```

