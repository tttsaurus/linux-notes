# Screenshot Alternative
Install flameshot
```bash
sudo dnf install flameshot
```
Add keybind (for Wayland: use `QT_QPA_PLATFORM=xcb` to bypass Wayland restrictions)
```
env QT_QPA_PLATFORM=xcb flameshot gui
```

