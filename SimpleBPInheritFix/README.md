[English](./README.md) | [中文](./README_CN.md)

# 📘 SimpleBPInheritFix Tutorial

**Supported Unreal Engine Versions**: 5.2–5.8

One-click fix for the Actor Blueprint inheritance component corruption bug.

> ⚠️ This plugin modifies multiple blueprints along the inheritance chain. Changes are conservative, but before committing to source control, please verify that the auto-checked-out files are the expected ones.

---

## 📝 How to Use

1. Install the plugin and compile successfully
2. In the Content Browser, **right-click any affected Actor Blueprint** (ancestor / self / descendant — all work equally)
3. Select **Component Repair → Fix Inheritance Pollution**
4. Open Output Log and filter by `LogSBIF` to monitor execution
5. The editor will show a save dialog — **save the dirtied blueprints** to finish

---

## ❓ FAQ

**Scan returns 0 findings but corruption clearly exists?**  
What you're hitting is probably not the issue this plugin covers (rare case). Paste the full `LogSBIF` log for diagnosis.

**What if CheckOut fails?**  
The plugin will continue with in-memory repair, but you'll need to handle checkout manually before saving (`p4 edit` / `git lfs lock` / `svn lock`). Common causes: files locked by others, server unreachable, or mapping doesn't include the file.

**Which blueprint should I pick as the anchor?**  
Pick any — ancestors, self, and descendants will all be scanned. The result is identical.

**Does it work without Perforce?**  
- Git: CheckOut is essentially a no-op, files are writable by default
- Git LFS: Issues an LFS lock request
- SVN: Runs `svn lock`
- No source control: The CheckOut stage is skipped entirely

---

## ⚠️ Limitations

- Only handles the specific component corruption this plugin covers; other blueprint issues are out of scope
- Components dynamically created in Construction Script are not scanned

---

## Version History

See [CHANGELOG.md](../CHANGELOG.md).

## Language

Open **Edit → Project Settings**, select the SimpleBPInheritFix settings page, and set **Language** to **English** or **Chinese**. English is the default. Available user-facing UI, tooltips, notifications, and functional logs update immediately without restarting the editor. Public API names and module startup/shutdown logs remain in English.

## Contact

For questions or feedback, contact me through any of the channels below. I will take care of it when I see your message.

- **Email:** [mzsh.me@icloud.com](mailto:mzsh.me@icloud.com)
- **WeChat:** `mengzhishanghun`
- **X:** [@mengzhishanghun](https://x.com/mengzhishanghun)
