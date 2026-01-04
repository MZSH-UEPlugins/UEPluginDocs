# 📘 Simple Input Simulator Plugin Tutorial (Blueprint Edition)

**Simple Input Simulator** is a lightweight input simulation plugin for Unreal Engine.
It provides **three Blueprint functions** to simulate keyboard, mouse, and gamepad inputs programmatically.

---

## 🔧 Plugin Usage

The plugin works immediately after enabling.
All functions are available in the **Simple Input Simulator** category in Blueprint.

---

## 🧠 Blueprint Node Interface

![All Available Functions](Images/ShowAllFunctions.png)

### 🎮 Key Input Simulation

| Node                | Description                                    |
|--------------------|------------------------------------------------|
| `Simulate Key Input` | Simulate press/release events for any input key |

**Parameters:**
- **Key**: Input key to simulate (Space Bar, Left Mouse Button, Gamepad buttons, etc.)
- **Pressed**: `true` for press, `false` for release
- **Player Index**: Target player controller (0 = first player)

![Key Input Usage](Images/ShowSimulateKeyInput.png)

### 🕹️ Axis Input Simulation

| Node                 | Description                                    |
|---------------------|------------------------------------------------|
| `Simulate Axis Input` | Simulate analog input values (movement, mouse delta, gamepad sticks) |

**Parameters:**
- **Key**: Axis input key (Mouse X/Y, Gamepad Left Thumbstick X/Y, etc.)
- **Axis Value**: 3D vector for input values (typically use X component)
- **Player Index**: Target player controller (0 = first player)

![Axis Input Usage](Images/ShowSimulateAxisInput.png)

### 🧹 Input State Management

| Node               | Description                          |
|-------------------|--------------------------------------|
| `Flush Pressed Keys` | Clear all pressed keys and reset input state |

**Parameters:**
- **Player Index**: Target player controller (0 = first player)

---

## 🔁 Enhanced Input Compatibility

| Input System     | Compatibility                                  |
|-----------------|------------------------------------------------|
| Legacy Input     | ✅ Full support                                |
| Enhanced Input   | ✅ Full support - triggers Input Actions automatically |

The plugin works at the raw input level, so both input systems receive simulated events automatically.

---

## 🧪 Blueprint Examples (Images)

### Key Input Example
![Key Input Usage](Images/ShowSimulateKeyInput.png)

Simulate pressing Space Bar for Player 0.

### Axis Input Example
![Axis Input Usage](Images/ShowSimulateAxisInput.png)

Simulate mouse movement or gamepad stick input.

---

## ✅ Tips & Notes

- All functions return `bool` indicating success/failure
- Player Index 0 targets the first player controller
- Use Flush Pressed Keys to prevent "stuck" key states
- Axis inputs automatically adapt to current frame rate
- Check Output Log for detailed simulation messages (LogSimpleInputSimulator)

---

## Support

For questions or feedback, please leave a comment on the Fab product page.

