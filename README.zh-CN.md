# Mac OSX Wi-Fi Location Changer

[English](README.md) | 简体中文

- 当连接到指定的 Wi-Fi（SSID）时，自动切换 macOS 网络位置
- 可以根据不同的 Wi-Fi SSID 使用不同的 IP 配置
- 支持在位置切换后触发外部脚本

## macOS 15.7.5 / Sequoia 用户先看这里

如果你使用的是 macOS Sequoia，尤其是 15.6+ 或 15.7.x，建议先看这一节，再决定如何安装或排障。

- **在 macOS 15+ 上，Shortcuts 已经是主 SSID 检测路径**。脚本现在使用 `shortcuts run ... --output-type public.plain-text --output-path ...`，不再只依赖 stdout，这在 `launchd` 下更稳。
- **快捷指令名称兼容性已经补上**。脚本同时接受 `Current Wi-Fi` 和旧名字 `Current WiFi`。
- **增加了尽力而为的多级回退**。如果 Shortcuts 不可用，脚本会按顺序尝试 `ipconfig getsummary`、`networksetup -getairportnetwork`、preferred networks 推断、`defaults read`、`system_profiler`。
- **配置 UI 现在能正确处理 macOS 15**。它不再把 Sequoia 误判成 macOS 26，映射计数更准确，可用 Location 显示更清楚，也不会再把 `<Unable to detect - Privacy Protected>` 这种占位文本保存成 SSID 映射。
- **LaunchAgent plist 已经清理过**。此前存在的无效 `WatchPaths` 条目已经移除。
- **这次不涉及任何编译型二进制修改**。当前项目仍然只是 shell 脚本加一个 LaunchAgent plist。

重要行为说明：

- 直接在仓库里运行 `bash ./locationchanger` 时，会使用仓库内的配置路径（`./locationchanger.conf`），但它仍然会去找系统里的特权 helper：`/usr/local/bin/locationchanger-helper`。
- 真正验证功能时，请先执行 `./install.sh` 完成安装，确认 sudoers 配置已加好，再测试 `/usr/local/bin/locationchanger` 或已加载的 LaunchAgent。

新版 Sequoia 的已知限制：

- 在部分 macOS 15.6+ / 15.7.x 系统上，Apple 对纯 shell 方式读取 SSID 的限制更严格。此时最可靠的短期方案是使用 Shortcuts 暴露当前 SSID。
- 如果 Shortcuts 能工作，而 shell 回退方法不行，项目在安装后通常仍然可以继续使用。
- 如果 Shortcuts 和 shell 回退都拿不到 SSID，那么纯脚本方案在这台机器上可能已经不够稳定，长期看需要改成带签名的 app/agent 方案。

## 快速开始

### 安装并配置 LocationChanger

1. 如果你使用的是 macOS 15+ / Sequoia，请先创建并测试 `Current Wi-Fi` 快捷指令：

   ```bash
   tmp=$(mktemp)
   shortcuts run "Current Wi-Fi" --output-type public.plain-text --output-path "$tmp"
   cat "$tmp"
   rm -f "$tmp"
   ```

   如果你之前创建的是 `Current WiFi`，当前脚本也兼容这个旧名字。

2. 安装 LocationChanger：

   ```bash
   ./install.sh
   ```

3. 配置 SSID 映射：

   ```bash
   ./config-ui.sh
   ```

   - 选择选项 1 添加 SSID 映射
   - 如果还没安装，选择选项 5 完成安装

4. 配置 sudoers。这一步是切换 Location 所必需的：

   ```bash
   sudo visudo
   ```

   添加下面这一行，并把 `your_username` 替换成你的实际用户名：

   ```text
   your_username ALL=(ALL) NOPASSWD: /usr/local/bin/locationchanger-helper
   ```

5. 通过切换不同 Wi-Fi 网络来测试已安装的服务。

## 配置 LocationChanger

最简单的配置方式是使用交互式配置工具：

```bash
./config-ui.sh
```

这个工具提供完整的命令行配置界面。具体说明请看 [使用配置 UI](#使用配置-ui)。

### macOS 通知

脚本在切换 Location 后会触发 macOS 通知。如果你不需要这个行为，删除以 `osascript` 开头的那几行即可。

## 安装 LocationChanger

### 推荐：使用配置工具

最简单的安装方式是通过配置工具：

```bash
./config-ui.sh
```

然后选择选项 5，**Install LocationChanger**。这会：

- 执行安装脚本
- 保留你已有的配置
- 提示后续步骤

### 自动安装

运行：

```bash
./install.sh
```

### 手动安装

复制这些文件：

```bash
cp locationchanger /usr/local/bin
cp locationchanger-helper /usr/local/bin
cp locationchanger.conf /usr/local/bin
cp LocationChanger.plist ~/Library/LaunchAgents/
```

给脚本加执行权限：

```bash
chmod +x /usr/local/bin/locationchanger
chmod +x /usr/local/bin/locationchanger-helper
sudo chown root /usr/local/bin/locationchanger-helper
sudo chmod 500 /usr/local/bin/locationchanger-helper
```

把 `LocationChanger.plist` 加载为 launchd 守护项：

```bash
launchctl load ~/Library/LaunchAgents/LocationChanger.plist
```

如果你把 `locationchanger` 放到别的位置，记得同步修改 `LocationChanger.plist` 里的路径。

## 查看日志

你可以在 `locationchanger` 大约第 12 行附近修改日志文件路径：

```bash
exec &>/usr/local/var/log/locationchanger.log
```

实时查看日志：

```bash
tail -f /usr/local/var/log/locationchanger.log
```

## 在位置切换后运行外部脚本

按照约定，如果你在这个目录下放一个可执行脚本，并命名为：

`locationchanger.callout.sh`

然后重新运行安装器，LocationChanger 每次切换 Location 时都会调用这个脚本。

## 测试 LocationChanger

建议先在当前环境里准备两个 Location，例如 `home` 和 `guest`，并分别映射到两个不同的 SSID，例如主 Wi-Fi 和访客 Wi-Fi。然后通过 Wi-Fi 菜单在这两个网络之间切换。

你可以用下面的命令查看日志中的成功或失败信息：

```bash
tail /usr/local/var/log/locationchanger.log
```

## 卸载 LocationChanger

### 推荐：使用配置工具

```bash
./config-ui.sh
```

选择选项 6，**Uninstall LocationChanger**，即可完整移除所有文件和配置。

### 手动卸载

```bash
# 停止服务
launchctl unload ~/Library/LaunchAgents/LocationChanger.plist

# 删除文件
sudo rm -f /usr/local/bin/locationchanger
sudo rm -f /usr/local/bin/locationchanger-helper
sudo rm -f /usr/local/bin/locationchanger.conf
sudo rm -f /usr/local/bin/locationchanger.conf.backup
sudo rm -f /usr/local/bin/locationchanger.callout.sh
rm -f ~/Library/LaunchAgents/LocationChanger.plist
sudo rm -f /usr/local/var/log/locationchanger.log

# 手动删除 sudoers 配置
sudo visudo
# 删除这一行：your_username ALL=(ALL) NOPASSWD: /usr/local/bin/locationchanger-helper
```

## macOS 兼容性与安全性

### 在 macOS Sequoia 15.0+ 上使用 Shortcuts

这个版本通过 Shortcuts app 支持 macOS Sequoia 15.0+。当传统方法失效时，脚本会使用快捷指令读取当前 Wi-Fi SSID；如果 Shortcuts 不可用，则自动回退到旧方法。

如果你在配置工具中看到 `Privacy Protected` 或 `<redacted>`，仍然可以手动配置 SSID 映射。如果要自动检测，请创建一个名为 `Current Wi-Fi` 的快捷指令。当前脚本也兼容旧名字 `Current WiFi`。

### 创建 `Current Wi-Fi` 快捷指令

打开 **Shortcuts** 应用，点击 **+** 新建快捷指令。在右侧面板中选择 **Device**，把 **Get Network Details** 拖到左侧。然后进入 **Scripting**，找到 **Stop and Output**，放到第一个动作下面。点击 **Play** 测试，确认它能显示当前 Wi-Fi 名称，然后将快捷指令保存为 **Current Wi-Fi**。

如果你已经创建的是 **Current WiFi**，当前脚本也能识别。

界面大致如下：

<img width="2446" height="1624" alt="Shortcuts App New shortcut 3" src="https://github.com/user-attachments/assets/9eae481c-4881-478b-a9a2-86c0363a955e" />

### 安全地切换 Location

从 macOS Sequoia 15.5 开始，切换网络位置需要管理员权限。这个项目使用了一个受限 helper 脚本来尽量降低安全风险。

安全收益：

- 只有指定的 helper 脚本可以免密码执行
- helper 脚本归 root 所有，并且权限受限（`500`）
- helper 脚本只允许切换网络位置，有助于避免权限提升
- 输入校验可以降低注入风险

### 自动回退到 `Automatic`

如果配置文件里没有匹配到 SSID，脚本会自动切换到 `Automatic` 这个 Location。这同样需要上面提到的 sudoers 配置。

## Configuration UI

项目附带一个命令行配置工具，用于管理 LocationChanger。

<img width="1066" height="862" alt="Bash Configuration UI" src="https://github.com/user-attachments/assets/9d32b9bb-62c0-4978-8b90-605222bc993c" />

### 使用配置 UI

```bash
./config-ui.sh
```

这个交互式工具提供以下功能：

#### 菜单选项

1. **Add SSID mapping**：添加新的 Wi-Fi 到 Location 的映射
2. **Remove SSID mapping**：删除已有映射
3. **Refresh current SSID**：刷新当前 Wi-Fi 状态
4. **View configuration file**：查看当前配置文件
5. **Install LocationChanger**：执行安装脚本并保留现有映射
6. **Uninstall LocationChanger**：完整移除所有文件和配置
7. **Exit**：退出配置工具

#### 功能特点

- 彩色终端界面，状态显示更直观
- 展示当前 Wi-Fi 状态和映射数量
- 安装时保留现有映射
- 支持完整卸载
- 带错误处理和直接反馈
- 对权限需求有感知

#### 使用示例

```bash
# 启动配置工具
./config-ui.sh

# 工具会显示：
# - 当前 Wi-Fi 状态
# - 现有映射
# - 交互式菜单
```

#### 通过 UI 安装

1. 运行 `./config-ui.sh`
2. 选择选项 5，**Install LocationChanger**
3. 工具会：
   - 检查已有映射
   - 运行安装脚本
   - 恢复你的配置
   - 提示 sudoers 的下一步操作

#### 通过 UI 卸载

1. 运行 `./config-ui.sh`
2. 选择选项 6，**Uninstall LocationChanger**
3. 确认删除
4. 工具会：
   - 停止服务
   - 删除所有文件
   - 清理配置
   - 提醒你手动清理 sudoers

## 手动配置

如果你更偏向手动配置，可以先从示例文件复制出配置文件：

```bash
cp ./locationchanger.conf.sample ./locationchanger.conf
```

然后为每一组 Location 和 SSID 添加一行。每一行包含一个 Location 名称和一个 Wi-Fi SSID，中间用空格分隔。大小写要完全一致，必要时使用引号。

例如：

```text
home myWifiName
```

如果你的 SSID 中间包含空格，请加引号：

```text
home "Wu Tang LAN"
```

请确保使用的 Location 名称和 **系统设置** → **网络** 中显示的名称完全一致，SSID 也必须和 Wi-Fi 菜单中的名称完全一致。大小写和空格都必须匹配。

## 必要设置

切换 Location 仍然需要手动完成以下设置：

1. 运行 `./install.sh` 安装 helper 脚本
2. 在 sudoers 文件中加入下面这一行：

   ```bash
   sudo visudo
   ```

   将 `your_username` 替换为你的实际用户名：

   ```text
   your_username ALL=(ALL) NOPASSWD: /usr/local/bin/locationchanger-helper
   ```

   你可以用 `whoami` 查看当前用户名。
