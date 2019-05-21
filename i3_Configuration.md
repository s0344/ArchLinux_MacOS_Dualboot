# Rising after i3 installation

A rough guide of rising i3 after installation. It will not be as detailed. Refer to the `dotfiles` repo for more info of specific file.

  - after i3 download
  - modify bash_profile
  - run `startx` to start i3
  - download `zathura` `yay` `compton` `rofi` `polybar`
  - modify .bashrc
  - modify .i3 config
  - modify sudoers
  - install driver (p for pacman, y for yay)
	> install y kbdlight, p xorg-xbacklight, xorg-xev (keys), xorg-xprop (window name)
  
  - Run this command to detect keys
  
 ```  xev | awk -F'[ )]+' '/^KeyPress/ { a[NR+2] } NR in a { printf "%-3s %s\n", $5, $8 }'```
	
  - chinese font

  ```sudo pacman -S wqy-zenhei (or wqy-bitmapfont)```

  - delete i3lock, get i3lock-color
    https://github.com/tungcena/tungcena-i3lock
  - install `scrot` and `xorg-xrandx` for i3lock
  - poly bar copy config
  - compton copy config
  - rofi copy config or run command
