# Screenshot Alternative
Install flameshot
```bash
sudo dnf install flameshot
```
Add keybind (for Wayland: use `QT_QPA_PLATFORM=xcb` to bypass Wayland restrictions)
```
env QT_QPA_PLATFORM=xcb flameshot gui
```

# Zen Browser `.desktop` Config
In case you installed the `tar.xz` version

```bash
sudo nano ~/.local/share/applications/zen.desktop
```

```
[Desktop Entry]
Name=Zen Browser
Exec=zen_runnable_path %U
Icon=zen_icon_path
Type=Application
Categories=Network;WebBrowser;
Terminal=false
StartupNotify=true
StartupWMClass=zen
MimeType=text/html;text/xml;application/xhtml+xml;x-scheme-handler/http;x-scheme-handler/https;
```

Parameters like `Terminal=false StartupNotify=true ...` prevent users to accidentally open multiple windows at once.

```bash
update-desktop-database ~/.local/share/applications
```

# GRUB Boot Menu Default Entry
```bash
sudo grub2-set-default <index>/"<entry name>"
```

Like 
```bash
sudo grub2-set-default 0
sudo grub2-set-default "Fedora ..."
```

# AppImage Usage
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
  Name=AppName
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

# Set `JAVA_HOME` and `PATH`
```bash
#!/usr/bin/env bash

echo "Available Java installations:"
echo

ls /usr/lib/jvm/

echo
read -p "Enter JDK directory name: " JDK_NAME

JDK_PATH="/usr/lib/jvm/$JDK_NAME"

if [ ! -d "$JDK_PATH" ]; then
    echo "JDK not found!"
    exit 1
fi

echo
echo "Setting JAVA_HOME and updating PATH..."

export JAVA_HOME="$JDK_PATH"
export PATH="$JAVA_HOME/bin:$PATH"

echo
echo "JAVA_HOME set to:"
echo "$JAVA_HOME"

echo
java -version
javac -version

sed -i '/JAVA_HOME/d' ~/.bashrc
sed -i '/\$JAVA_HOME\/bin/d' ~/.bashrc

echo "export JAVA_HOME=\"$JDK_PATH\"" >> ~/.bashrc
echo 'export PATH="$JAVA_HOME/bin:$PATH"' >> ~/.bashrc

echo
echo "Saved to ~/.bashrc"
echo "Open a new terminal or run:"
echo "source ~/.bashrc"
```
```bash
sudo chmod +x set_java.sh
```
```bash
source ./set_java.sh
```
