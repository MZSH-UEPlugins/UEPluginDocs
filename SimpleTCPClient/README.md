[English](./README.md) | [中文](./README_CN.md)

# 📘 SimpleTCPClient Plugin Tutorial (Blueprint Edition)

**SimpleTCPClient** is a lightweight TCP client plugin designed for Unreal Engine.  
It supports **runtime dynamic configuration of client channels**, and all functionality is exposed to Blueprint for easy integration.

## Compatibility

- Unreal Engine 5.2–5.8
- Version 1.1.0 (Version 2)
- [Changelog](../CHANGELOG.md)

---

## 🔧 Plugin Initialization

The plugin is automatically activated after being enabled.  
It registers internally as a `GameInstanceSubsystem`, requiring no manual startup or shutdown code.

---

## ⚙️ Static Channel Configuration (Optional)

You can predefine client channels via **Project Settings → SimpleTCPClient Settings**.  
It is recommended to configure channels here for easier management.

### 🔹 Client Channel Config Fields

| Field                 | Example             | Description                                |
|----------------------|---------------------|--------------------------------------------|
| Channel Name          | `DefaultClient`     | Name used in Blueprint to reference channel|
| Remote Address (IP:Port) | `127.0.0.1:8888` | The target address to connect              |
| Auto Connect          | `True`              | Whether to auto-connect on startup         |
| Auto Reconnect Interval | `1.0` (seconds)   | Interval between reconnection attempts     |
| Max Receive Bytes     | `1024`              | Max size per receive                       |

---

## 🧠 Blueprint Node Interface

### 📥 Receiving

| Node Function              | Description                             |
|---------------------------|-----------------------------------------|
| `BindMessageHandle`       | Bind a message receive delegate         |
| `UnbindMessageHandle`     | Unbind a specific delegate              |
| `UnbindAllMessageHandle`  | Unbind all delegates and close socket   |

### 📤 Sending

| Node Function   | Description                                   |
|-----------------|-----------------------------------------------|
| `SendMessage`   | Send a byte array through a specific channel  |

### ⚙️ Runtime Channel Management

| Node Function              | Description                                         |
|---------------------------|-----------------------------------------------------|
| `StartConnection`         | Manually start a connection (for non-auto channels) |
| `StopConnection`          | Stop and destroy the socket                         |
| `IsConnected`             | Check whether a channel is currently connected      |
| `UpdateTCPClientConfig`   | Create or update channel config (auto reconnects)   |
| `GetTCPClientConfig`      | Get the current configuration of a channel          |

---

## 🔁 Socket Lifecycle

| Type         | Created When                           | Destroyed When                          |
|--------------|-----------------------------------------|------------------------------------------|
| Client Socket| When calling `StartConnection` or auto  | When calling `StopConnection` or shutdown|

---

## 🧪 Blueprint Examples (Images)

### All Blueprint Nodes  
![ShowAllFunction](./Images/ShowAllFunction.jpg)

### Manual Start/Stop Connection  
![Connection](./Images/StartConnection%20And%20StopConnection.jpg)  
Note: If the channel is set to auto-connect and has a reconnect interval configured, manual startup is not required.

### Bind & Unbind Message Handler  
![ReceiveMessage](./Images/ReceiveMessage.jpg)

### Dynamically Update Client Channel Config  
![DynamicUpdateChannel](./Images/DynamicUpdateChannel.jpg)  
Note: The message delegate remains valid across socket reconnections. The update order does not matter — once config is updated, callbacks will still be triggered.

### Send Message (String or Structured Data)  
![SendMessage](./Images/SendMessage.jpg)

---

## ✅ Tips & Notes

- Only IPv4 is supported — no domain name, IPv6, or encryption
- All sockets are fully managed internally, no manual cleanup needed
- The plugin lives within the `GameInstance` lifecycle, and is not affected by level transitions

---

## 🔧 Recommended Companion Plugin

Use with **SimpleByteConversion** to easily convert common Unreal types (FString, float, int, etc.) to/from `TArray<uint8>` and build structured protocols.

---

## Version History

See [CHANGELOG.md](../CHANGELOG.md).

## Language

Open **Edit → Project Settings**, select the SimpleTCPClient settings page, and set **Language** to **English** or **Chinese**. English is the default. Available user-facing UI, tooltips, notifications, and functional logs update immediately without restarting the editor. Public API names and module startup/shutdown logs remain in English.

## Support

For questions or feedback, contact me through any of the channels below. I will take care of it when I see your message.

- **Email:** [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com)
- **WeChat:** `mengzhishanghun`
- **X:** [@mengzhishanghun](https://x.com/mengzhishanghun)

