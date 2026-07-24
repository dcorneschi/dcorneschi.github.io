# PuTTY Default Settings

<img src="/articles/images/putty-logo.svg" alt="Putty" width="100">

Configuration changes to apply to the **Default Settings** profile. These settings persist for all new sessions.

> Open PuTTY → load **Default Settings** → make changes → click **Save** (back on the Session panel) to persist them.

### My Default Settings

| Setting | Value |
|---------|-------|
| Font | Lucida Console, size 10 |
| Bell | Disabled |
| Window size | 120 x 40 |
| Scrollback | 10000 lines |
| Scrollbar | Hidden |
| Color scheme | Gotham |

---

## Font

**Window → Appearance → Font settings**

| Setting | Value |
|---------|-------|
| Font | Lucida Console |
| Size | 10 (adjust to preference) |

Click **Change...** next to the font display, select **Lucida Console**, set the size, and confirm.

---

## Disable Bell

**Terminal → Bell**

| Setting | Value |
|---------|-------|
| Action to happen when a bell occurs | None (bell disabled) |

This prevents the annoying system beep on tab completion failures or when hitting the end of scrollback.

---

## Window Size

**Window**

| Setting | Value |
|---------|-------|
| Columns | 120 |
| Rows | 40 |

Adjust to fit your monitor. 120x40 works well on 1920x1080 displays.

---

## Scrollback

**Window**

| Setting | Value |
|---------|-------|
| Lines of scrollback | 10000 |
| Display scrollbar | Never |

10,000 lines ensures you can scroll back through long command outputs without losing history.

---

## Color Scheme

**Window → Colours**

PuTTY with Gotham color scheme and Lucida Console font

<img src="/articles/images/putty-gotham-lucida.png" alt="Putty" width="800"/>

A dark theme that's easy on the eyes. Adjust RGB values to your preference, or use a pre-built theme from [putty-color-themes](https://github.com/AlexAkulov/putty-color-themes) — a collection of ready-to-import `.reg` files for popular color schemes. For more inspiration, browse [iterm2colorschemes.com](https://iterm2colorschemes.com) which previews hundreds of terminal color schemes.

---

## Registry Export (Backup/Restore)

PuTTY stores sessions in the Windows Registry. You can export and import settings to avoid manual reconfiguration:

### Export

```cmd
reg export "HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions\Default%%20Settings" putty-defaults.reg
```

### Import (after reinstall)

```cmd
reg import putty-defaults.reg
```

> `%%20` represents a space in the registry key name. The default session is stored as `Default%20Settings`.

### Export All Sessions

```cmd
reg export "HKEY_CURRENT_USER\Software\SimonTatham\PuTTY" putty-all-sessions.reg
```

---

## Quick Checklist

After a fresh Windows install:

1. Open PuTTY, load **Default Settings**
2. **Window → Appearance** → Change font to Lucida Console, size 10
3. **Terminal → Bell** → Set to None
4. **Window** → Set columns 120, rows 40, scrollback 10000
5. **Window → Colours** → Apply color scheme
6. Go back to **Session** → Click **Save** on Default Settings
7. (Optional) Export registry key for future reinstalls

---

## References

- [PuTTY Documentation](https://www.chiark.greenend.org.uk/~sgtatham/putty/docs.html)
- [Solarized Color Scheme](https://ethanschoonover.com/solarized/)
