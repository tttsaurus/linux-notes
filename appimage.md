- Download `.AppImage`
- Grant Access
  ```bash
  chmod +x *.AppImage
  ```
- Move it elsewhere
  ```bash
  mv *.AppImage ~/Applications/
  ```
  Make dir first if needed
  ```bash
  mkdir -p ~/Applications
  ```
- Create `*.desktop`
  ```bash
  sudo nano ~/.local/share/applications/*.desktop
  ```
- ```                              
  [Desktop Entry]
  Name=Chatbox
  Exec=*.AppImage
  Icon=utilities-terminal
  Type=Application
  Categories=Utility;
  ```
- ```bash
  update-desktop-database ~/.local/share/applications
  ```
- Install fuse if needed
  ```bash
  sudo dnf install fuse fuse-libs
  ```
