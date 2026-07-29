[English](./README.md) | [中文](./README_CN.md)

# SimpleSSHTunnel 用户指南

## 兼容性

- Unreal Engine 5.2–5.8
- 版本 1.1.0（Version 2）
- [更新日志](../CHANGELOG.md)

## 1. 配置 SSH 连接

使用 SimpleSSHTunnel 插件前，在 **项目设置 → Engine → Simple SSH Tunnel** 中配置 SSH 连接。

### 1.1 访问 SSH 设置

1. 在 Unreal Engine 编辑器中打开 **项目设置**。
2. 在左侧面板导航至 **Engine → Simple SSH Tunnel**。
3. 配置以下参数：

   - **Local Address**
   - **Local Port**
   - **Remote Server Address**
   - **Remote Port**
   - **SSH Server Address**
   - **SSH Port**（默认：22）
   - **SSH Username**
   - **SSH Password**

可选：**Subsystem Class** 允许你覆盖默认行为。

所有设置保存在 `DefaultSimpleSSHTunnel.ini` 中，引擎启动时自动应用。

---

## 2. 蓝图 API

配置 SSH 连接后，以下**四个蓝图函数**可用：

### 2.1 创建 SSH 隧道

```blueprint
Create SSHTunnel (Tunnel Name: String) → Return Value: Bool
```

使用指定名称创建新的 SSH 隧道。

### 2.2 检查 SSH 隧道状态

```blueprint
Check SSHTunnel Is Running (Tunnel Name: String) → Return Value: Bool
```

检查指定的 SSH 隧道是否正在运行。

### 2.3 关闭指定 SSH 隧道

```blueprint
Close SSHTunnel (Tunnel Name: String) → Return Value: Bool
```

关闭指定的 SSH 隧道。

### 2.4 关闭所有 SSH 隧道

```blueprint
Close All SSHTunnel () → Return Value: Bool
```

关闭所有活跃的 SSH 隧道。

📌 **注意**：
- 默认子系统在引擎关闭时自动关闭所有隧道。
- 可通过指定自定义 **Subsystem Class** 来定义自定义行为。

---

## 3. 示例工作流

1. 在 **项目设置** 中配置 SSH 隧道。
2. 在蓝图中调用 **Create SSHTunnel** 创建隧道。
3. 使用 **Check SSHTunnel Is Running** 验证状态。
4. 如需关闭连接，调用 **Close SSHTunnel** 或 **Close All SSHTunnel**。

---

## 4. 补充说明

- 完全用 C++ 实现，完整蓝图集成。
- 引擎关闭时自动清理 SSH 隧道（除非自定义）。

---

## 版本历史

参见 [CHANGELOG.md](../CHANGELOG.md)。

## 语言

打开 **编辑 → 项目设置**，进入 SimpleSSHTunnel 设置页，将 **Language** 设置为 **English** 或 **中文**。默认语言为英文。面向用户的界面、提示、通知和功能日志会立即更新，无需重启编辑器；公开 API 名称以及模块启动/关闭日志始终保持英文。

## 支持

如有问题、反馈或希望加入 UE 技术交流群，请访问以下统一联系方式页面。

- **联系入口：** [邮箱、微信群和 X](https://mengzhishanghun.github.io/mengzhishanghun/contact/)
