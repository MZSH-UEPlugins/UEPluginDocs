[English](./README.md) | [中文](./README_CN.md)

# 📘 SimpleBPInheritFix 教程

**支持的虚幻引擎版本**：5.2–5.8

一键修复 Actor 蓝图继承链组件污染 Bug。

> ⚠️ 此插件会修改继承链上的多个蓝图。修改是保守的，但在提交到版本控制前，请确认自动签出的文件是否符合预期。

---

## 📝 使用方法

1. 安装插件并编译成功
2. 在内容浏览器中，**右键任何受影响的 Actor 蓝图**（祖先/自身/后代均可）
3. 选择 **Component Repair → Fix Inheritance Pollution**
4. 打开输出日志并过滤 `LogSBIF` 以监控执行
5. 编辑器会弹出保存对话框——**保存标记为脏的蓝图**即可完成

---

## ❓ 常见问题

**扫描返回 0 个发现但明显存在损坏？**  
你遇到的可能不是此插件覆盖的问题（罕见情况）。请粘贴完整的 `LogSBIF` 日志以供诊断。

**CheckOut 失败怎么办？**  
插件会继续进行内存中的修复，但你需要在保存前手动处理签出（`p4 edit` / `git lfs lock` / `svn lock`）。常见原因：文件被他人锁定、服务器不可达、或映射未包含该文件。

**应该选择哪个蓝图作为锚点？**  
随意选择——祖先、自身和后代都会被扫描，结果相同。

**没有 Perforce 也能用吗？**  
- Git：CheckOut 本质上是空操作，文件默认可写
- Git LFS：发起 LFS lock 请求
- SVN：运行 `svn lock`
- 无版本控制：跳过 CheckOut 阶段

---

## ⚠️ 限制

- 仅处理此插件覆盖的特定组件污染问题；其他蓝图问题不在范围内
- 不扫描在 Construction Script 中动态创建的组件

---

## 版本历史

参见 [CHANGELOG.md](../CHANGELOG.md)。

## 语言

打开 **编辑 → 项目设置**，进入 SimpleBPInheritFix 设置页，将 **Language** 设置为 **English** 或 **中文**。默认语言为英文。面向用户的界面、提示、通知和功能日志会立即更新，无需重启编辑器；公开 API 名称以及模块启动/关闭日志始终保持英文。

## 联系

如有问题、反馈或希望加入 UE 技术交流群，请访问以下统一联系方式页面。

- **联系入口：** [邮箱、微信群和 X](https://mengzhishanghun.github.io/mengzhishanghun/contact/)
