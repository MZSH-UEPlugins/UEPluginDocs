[English](./README.md) | [中文](./README_CN.md)

# 📘 SimpleWebSocket 用户指南

本指南介绍如何配置和使用 SimpleWebSocket 插件在 Unreal Engine 项目中进行 WebSocket 通信。

## 兼容性

- Unreal Engine 5.2–5.8
- 版本 1.1.0（Version 2）
- [更新日志](../CHANGELOG.md)

---

## 🛠️ 插件设置

启用插件后，前往项目设置面板的 `Simple WebSocket Settings`。

### 步骤：

1. 打开 Unreal Engine 编辑器  
2. 前往 `编辑 > 项目设置`  
3. 向下滚动找到 `Simple WebSocket Settings`  
4. 添加新的 WebSocket 连接条目（如命名为 `Test`）  
5. 输入 WebSocket 地址（推荐格式：`ws://127.0.0.1:xxxx`，**不要使用 localhost**）  
6. 启用 **Auto Connect**（可选；启用后运行时自动建立连接）

📷 示例截图：

![WebSocket 设置](./Images/20250524103328.png)

---

## 🎮 蓝图使用示例

可使用 GameInstance 级别的蓝图函数来连接、发送、接收和关闭 WebSocket 连接。所有函数都是静态的，可从任何蓝图调用。

### 可用节点：

- `Connect Web Socket`：按名称连接 WebSocket  
- `Send WebSocket Message`：发送文本消息  
- `Bind Web Socket Message`：绑定蓝图事件接收消息（支持多个回调）  
- `Close Web Socket`：关闭指定连接  
- `Close All Web Socket`：关闭所有连接  
- `Check Web Socket Connect`：检查连接是否已建立  
- `Get Web Socket Config`：获取指定连接的配置  
- `Unbind Web Socket Message`：解绑消息事件  

📷 蓝图示例：

![蓝图示例](./Images/20250524104722.png)

---

## ✅ 注意事项

- 插件仅支持 `ws://` 协议。不支持安全连接 `wss://`。
- 已在**打包构建**中完全测试——在生产环境中正常工作。
- WebSocket URL 中避免使用 `localhost`。始终使用 `127.0.0.1` 以防止连接问题。
- 每个连接名称可以绑定多个回调，支持模块化事件处理。

---

## 版本历史

参见 [CHANGELOG.md](../CHANGELOG.md)。

## 语言

打开 **编辑 → 项目设置**，进入 SimpleWebSocket 设置页，将 **Language** 设置为 **English** 或 **中文**。默认语言为英文。面向用户的界面、提示、通知和功能日志会立即更新，无需重启编辑器；公开 API 名称以及模块启动/关闭日志始终保持英文。

## 支持

如有问题、反馈或希望加入 UE 技术交流群，请访问以下统一联系方式页面。

- **联系入口：** [邮箱、微信群和 X](https://mengzhishanghun.github.io/mengzhishanghun/contact/)
