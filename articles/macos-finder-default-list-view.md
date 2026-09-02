# Making List View the Default in macOS Finder

By default, macOS Finder opens folders in whatever view was last used, and new folders often default to icon view. If you prefer List view (or Column, Gallery, etc.), you can make it stick — for a single folder, for all folders, or for an entire folder tree. This guide covers all three.

## Set List View as the Global Default

This applies List view to every folder that doesn't already have its own saved view setting.

1. Open any folder in Finder.
2. Switch to List view — press `⌘2`, or use **View → as List**.
3. Open View Options — press `⌘J` (or **View → Show View Options**).
4. Adjust any other preferences you want as the default: sort order, text/icon size, and which columns are shown.
5. Click **Use as Defaults** at the bottom of the View Options panel.

The four Finder view shortcuts:

| View | Shortcut |
|------|----------|
| Icon | `⌘1` |
| List | `⌘2` |
| Column | `⌘3` |
| Gallery | `⌘4` |

## Set the View for Just One Folder

If you only want a specific folder to remember List view (without changing the global default):

1. Open that folder in Finder.
2. Switch to List view (`⌘2`).
3. Open View Options (`⌘J`) and adjust settings as desired.
4. **Do not** click "Use as Defaults" — just close the View Options panel.

Finder saves that folder's view preference on its own, so it reopens in List view next time.

## Apply View Settings to All Subfolders

To push one folder's view settings down through its entire subtree:

1. Open the parent folder in Finder and set it to List view.
2. Open View Options with `⌘J`.
3. Hold the **Option** (`⌥`) key — the **Use as Defaults** button changes to **Apply to all**.
4. Click **Apply to all** to apply the current view settings to every subfolder inside.

This is the quickest way to normalize a whole project or media directory to one consistent view.

## How macOS Stores These Settings

Per-folder Finder view preferences are saved in a hidden `.DS_Store` file inside each folder. That's why:

- Moving or deleting a folder's `.DS_Store` resets its custom view back to the default.
- Folders without a `.DS_Store` fall back to the global default set via **Use as Defaults**.
- `.DS_Store` files frequently show up in Git repos and shared drives — a common annoyance when a folder has custom view settings.

If you want to keep these files out of version control, add `.DS_Store` to your `.gitignore`.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| A folder ignores the global default | It has its own saved `.DS_Store` view | Re-set it per-folder, or delete that folder's `.DS_Store` |
| "Use as Defaults" didn't affect existing folders | It only applies to folders without their own settings | Use **Option → Apply to all** on the parent, or reset per-folder |
| Settings won't persist on a network/external drive | `.DS_Store` writes may be disabled or restricted | Check drive permissions; some setups block `.DS_Store` creation |
| Can't find "Apply to all" | Option key not held while View Options is open | Open `⌘J` first, then press and hold `⌥` |
