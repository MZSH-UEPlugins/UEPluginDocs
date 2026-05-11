[English](./README.md) | [中文](./README_CN.md)

# 📘 SimpleByteConversion 插件教程（蓝图版）

**SimpleByteConversion** 是一个为 Unreal Engine 蓝图系统设计的轻量级字节数据转换插件。  
支持**原生类型和结构体与字节/Json 之间的双向转换**，所有功能均以蓝图节点暴露。

---

## 🔧 插件初始化

插件启用后自动生效。  
所有功能通过蓝图节点访问，无需管理子系统或插件生命周期。

---

## 🔣 支持的类型转换

### 📦 原生类型 ↔ 字节数组

| 类型         | 转换支持 |
|-------------|---------|
| `int32`     | 转为/转自字节 |
| `int64`     | 转为/转自字节 |
| `float`     | 转为/转自字节 |
| `double`    | 转为/转自字节 |
| `bool`      | 转为/转自字节 |
| `FString`   | 转为/转自字节（UTF-8 编码）|

这些函数适用于自定义协议、轻量存储或二进制通信。

### 蓝图示例：

(插入图片: BasicTypeConversion.jpg)

---

## 🧱 结构体 ↔ 字节数组 / Json 字符串

插件支持任何暴露给蓝图的 `USTRUCT`：

- 与字节数组互转：`Struct To Bytes` / `Bytes To Struct`
- 与 JSON 字符串互转：`Struct To JsonString` / `JsonString To Struct`

此功能使用 Unreal 的反射系统自动序列化和反序列化字段，同时保留原始命名和浮点精度。

> ⚠️ 不支持：包含对象引用的结构体，如 `UObject*`、`TSoftObjectPtr`、`TWeakObjectPtr`，或资源句柄如 `FSoftObjectPath`。

### JSON 转换优势：

| 特性              | SimpleByteConversion                | UE 原生 FJsonObjectConverter |
|------------------|-------------------------------------|-------------------------------|
| 字段名保留        | ✅ 是                              | ❌ 转为 camelCase             |
| Float/Double 精度 | ✅ float(9位), double(17位)         | ❌ 默认精度较低               |
| 完整蓝图支持      | ✅ 是                              | ⚠️ 部分支持 / 需要 C++        |

---

## 🧪 蓝图节点概览（图片）

- 所有节点均为 `BlueprintPure`——无需执行引脚
- 可轻松组合用于网络、文件存取或自定义协议

![ShowAllFunction](./Images/ShowAllFunction.png)

---

### 蓝图示例：

![Test](./Images/20250502134059.png)

![Test](./Images/20250502134105.png)

![Test](./Images/20250719003048.png)

---

## ✅ 提示

- `StringToBytes` 和 `BytesToString` 使用 UTF-8 编码，确保跨平台安全转换
- 结合使用 `StructToBytes` 和 `StructToJsonString` 可用于调试和网络消息

---

## 🔧 推荐配套插件

与 **SimpleTCPClient** 和 **SimpleUDP** 插件配合使用效果极佳。  
一起使用可通过 TCP/UDP 发送结构化消息并轻松解码。

---

## 支持

如有问题或反馈，请在 Fab 产品页面留言。
