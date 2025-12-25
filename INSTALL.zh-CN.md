# 指纹驱动安装指南

## 支持的设备

- FPC 指纹传感器控制器 (USB ID: 10a5:9201)
- 红米 RedmiBook 15 2022 及兼容笔记本

## 环境要求

- Fedora Linux（已测试）或 Ubuntu
- CMake, GCC/G++
- libusb-1.0, libevent, dbus, openssl, opencv

## 安装步骤

### 1. 安装依赖（Fedora）

```bash
sudo dnf install libusbx-devel libevent-devel dbus-devel openssl-devel opencv-devel cmake gcc gcc-c++
```

### 2. 克隆并编译

```bash
git clone https://github.com/vrolife/fingerprint-ocv
cd fingerprint-ocv
git submodule init              # 初始化子模块
git submodule update            # 下载子模块代码
cmake -S . -B build             # 配置 CMake
cmake --build build -j$(nproc)  # 并行编译（使用所有 CPU 核心）
sudo cp build/src/fingerprint-ocv /usr/local/bin/  # 安装到系统路径
```

### 3. 禁用系统 fprintd

> ⚠️ **影响**：此操作会使系统自带的指纹驱动无法自动启动，但你可以随时恢复。

```bash
# 重命名 D-Bus 服务文件，防止自动启动
sudo mv /usr/share/dbus-1/system-services/net.reactivated.Fprint.service \
       /usr/share/dbus-1/system-services/net.reactivated.Fprint.service.disabled
```

### 4. 配置 D-Bus 权限

> ℹ️ **影响**：允许我们的自定义驱动获取 `net.reactivated.Fprint` D-Bus 名称。

```bash
sudo cat > /etc/dbus-1/system.d/net.reactivated.Fprint.conf << 'EOF'
<!DOCTYPE busconfig PUBLIC
 "-//freedesktop//DTD D-BUS Bus Configuration Version 1.0"
 "http://www.freedesktop.org/standards/dbus/1.0/busconfig.dtd">
<busconfig>
  <policy user="root">
    <allow own="net.reactivated.Fprint"/>
    <allow send_destination="net.reactivated.Fprint"/>
  </policy>
  <policy context="default">
    <allow own="net.reactivated.Fprint"/>
    <allow send_destination="net.reactivated.Fprint"/>
  </policy>
</busconfig>
EOF
```

### 5. 设置 systemd 服务

> ✅ **效果**：开机自动启动指纹驱动，无需手动运行。

```bash
sudo cat > /etc/systemd/system/fingerprint-ocv.service << 'EOF'
[Unit]
Description=Fingerprint Driver
After=dbus.service

[Service]
Type=simple
ExecStart=/usr/local/bin/fingerprint-ocv --bus=system
Environment=OPENSSL_CONF=/dev/null
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload        # 重新加载 systemd 配置
sudo systemctl enable --now fingerprint-ocv  # 启用并立即启动服务
```

### 6. 配置 GDM 登录界面指纹认证（可选，谨慎配置）

> ⚠️ **注意**：PAM 配置错误会导致无法登录！请先测试命令是否正确。

**推荐：简单安全的方式（指纹可选）**

```bash
sudo cp /etc/pam.d/password-auth /etc/pam.d/gdm-password
# 先确保能正常密码登录，再添加指纹
```

**进阶：添加指纹认证**

如果需要指纹，在 `/etc/pam.d/gdm-password` 开头添加一行：

```bash
auth        [success=done user_known=ignore new_authtok_reqd=ignore default=ignore]    pam_fprintd.so max_tries=3 timeout=5
```

完整示例（谨慎使用）：

```bash
sudo cat > /etc/pam.d/gdm-password << 'EOF'
auth        [success=done user_known=ignore new_authtok_reqd=ignore default=ignore]    pam_fprintd.so max_tries=3 timeout=5
auth        substack      password-auth
auth        optional      pam_gnome_keyring.so
auth        include       postlogin

account     required      pam_nologin.so
account     include       password-auth

password    substack       password-auth
-password   optional      pam_gnome_keyring.so use_authtok

session     required      pam_selinux.so close
session     required      pam_loginuid.so
session     required      pam_selinux.so open
session     optional      pam_keyinit.so force revoke
session     required      pam_namespace.so
session     include       password-auth
session     optional      pam_gnome_keyring.so auto_start
session     include       postlogin
EOF
```

> ✅ **效果**：密码登录优先，指纹作为可选。

### 7. 验证安装

```bash
systemctl status fingerprint-ocv    # 查看服务运行状态
fprintd-verify                      # 验证指纹识别是否正常
```

## 故障排除

### "dbus_bus_request_name failed: Request to own name refused by policy"

- 执行步骤 3 禁用系统 fprintd
- 执行步骤 4 配置 D-Bus 权限
- 重启 D-Bus：`sudo systemctl restart dbus-broker-service`

### "usbi_mutex_lock: Assertion failed"

- 使用 `OPENSSL_CONF=/dev/null` 环境变量运行驱动

### TLS/SSL 连接错误

- 在 systemd 服务中设置 `OPENSSL_CONF=/dev/null`

## 常用命令

| 命令 | 说明 |
|------|------|
| `lsusb \| grep 10a5` | 检查指纹硬件是否被系统识别 |
| `systemctl status fingerprint-ocv` | 查看驱动服务状态（active (running) 表示正常运行） |
| `fprintd-enroll` | 录入新的指纹（会引导你完成多次按压） |
| `fprintd-verify` | 验证已录入的指纹是否可用 |
| `sudo systemctl restart fingerprint-ocv` | 🔄 重启驱动（修改配置后需要执行） |
| `sudo systemctl stop fingerprint-ocv` | ⏹️ 停止驱动（停止后指纹功能不可用） |

## 恢复系统 fprintd（如需）

> ⚠️ **影响**：切换回系统自带的指纹驱动，之前的指纹数据不会丢失。

```bash
# 恢复系统 fprintd D-Bus 服务文件
sudo mv /usr/share/dbus-1/system-services/net.reactivated.Fprint.service.disabled \
       /usr/share/dbus-1/system-services/net.reactivated.Fprint.service

# 禁用自定义驱动服务
sudo systemctl disable fingerprint-ocv

# 重启系统 fprintd
sudo systemctl restart fprintd
```

## 快速参考

```bash
# ✅ 安装完成后验证
systemctl status fingerprint-ocv   # 应显示: active (running)

# ✅ 测试指纹
fprintd-verify                     # 扫描指纹，检查是否匹配成功

# 🔄 修改配置后
sudo systemctl daemon-reload && sudo systemctl restart fingerprint-ocv

# 🛑 出问题时停止
sudo systemctl stop fingerprint-ocv
```
