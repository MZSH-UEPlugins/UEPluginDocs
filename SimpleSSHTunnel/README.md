# SimpleSSHTunnel User Guide

## 1. Configure SSH Connection

To use the SimpleSSHTunnel plugin, configure the SSH connection in **Project Settings → Engine → Simple SSH Tunnel**.

### 1.1 Access SSH Settings

1. Open **Project Settings** in Unreal Engine editor.
2. Navigate to **Engine → Simple SSH Tunnel** in the left panel.
3. Configure the following parameters:

   - **Local Address**
   - **Local Port**
   - **Remote Server Address**
   - **Remote Port**
   - **SSH Server Address**
   - **SSH Port** (default: 22)
   - **SSH Username**
   - **SSH Password**

Optional: **Subsystem Class** allows you to override the default behavior.

All settings are saved in `DefaultSimpleSSHTunnel.ini` and automatically applied at engine startup.

---

## 2. Blueprint API

After configuring the SSH connection, the following **four Blueprint functions** are available:

### 2.1 Create SSH Tunnel

```blueprint
Create SSHTunnel (Tunnel Name: String) → Return Value: Bool
```

Creates a new SSH tunnel with the specified name.

### 2.2 Check SSH Tunnel Status

```blueprint
Check SSHTunnel Is Running (Tunnel Name: String) → Return Value: Bool
```

Checks if the specified SSH tunnel is running.

### 2.3 Close Specific SSH Tunnel

```blueprint
Close SSHTunnel (Tunnel Name: String) → Return Value: Bool
```

Closes the specified SSH tunnel.

### 2.4 Close All SSH Tunnels

```blueprint
Close All SSHTunnel () → Return Value: Bool
```

Closes all active SSH tunnels.

📌 **Note**:
- The default subsystem automatically shuts down all tunnels when the engine closes.
- You may define custom behavior by specifying a custom **Subsystem Class**.

---

## 3. Example Workflow

1. Configure the SSH tunnel in **Project Settings**.
2. In Blueprint, call **Create SSHTunnel** to create a tunnel.
3. Use **Check SSHTunnel Is Running** to verify its status.
4. Call **Close SSHTunnel** or **Close All SSHTunnel** to shut down connections if needed.

---

## 4. Additional Notes

- Fully implemented in C++ with full Blueprint integration.
- Automatically cleans up SSH tunnels when the engine shuts down (unless customized).

---

For feedback or support, please contact: `mengzhishanghun@outlook.com`

 
