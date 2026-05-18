[English](./README.md) | [中文](./README_CN.md)

# Simple AI DataAsset

DataAsset 与 JSON 文件自动双向绑定，支持 AI 直接修改 DataAsset。

## 功能特性

- **双向同步**：DataAsset 属性变更自动导出为 JSON；JSON 文件变更自动导入回 DataAsset
- **初始同步**：编辑器启动时比较时间戳，以最新版本为准进行同步
- **孤立清理**：自动删除对应 DataAsset 已不存在的 JSON 文件
- **版本控制集成**：保存前自动通过 Perforce/Git/SVN 检出文件
- **防循环机制**：内置防止无限同步循环的机制
- **高精度输出**：JSON 导出使用完整精度的浮点数值
- **属性名还原**：JSON 字段名与编辑器显示一致，而非 UE 内部的 camelCase 形式
- **智能写入**：仅在内容真正变化时才写入文件，避免版本控制中产生无意义的变更
- **编辑器撤销**：JSON 导入创建编辑器事务，支持 Ctrl+Z 撤销

## 安装

1. 将 `SimpleAIDataAsset` 文件夹复制到项目的 `Plugins/` 目录
2. 重启 Unreal 编辑器
3. 如未自动启用，在 编辑 > 插件 中启用

## 配置

打开 **项目设置 > 插件 > SimpleAIDataAsset**：

| 设置项 | 默认值 | 说明 |
|--------|--------|------|
| JSON Output Directory | `Saved/SimpleAIDataAsset` | JSON 文件存储目录（相对于项目根目录） |
| Enable Auto Sync | `true` | 开启/关闭自动同步 |

## 工作原理

1. 编辑器中修改 DataAsset 属性时，插件自动导出为 JSON 文件
2. 外部修改 JSON 文件（如 AI 工具）时，插件自动将变更导入回 DataAsset
3. 启动时，插件根据文件时间戳执行初始同步

## 支持的引擎版本

- Unreal Engine 5.2+

## 联系方式

如有问题或反馈，请在 Fab 产品页面留言。
