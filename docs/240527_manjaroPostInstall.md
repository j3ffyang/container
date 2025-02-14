# Manjaro Post-Install

#### System Settings

```sh
Setting > Search > App Search > disable
Setting > Privacy & Security > File History > disable
```

#### Installing Packages with `pacman`
```sh
terminator
veracrypt & veracrypt-console-bin (for UI)
oh-my-zsh
vim
openssh
jdk11-openjdk
graphviz
firewalld
neofetch
docker
chromium
vlc
geeqie
gnome-screenshot
fcitx-googlepinyin fcitx-im  fcitx-configtool
adobe-source-han-sans-otc-fonts adobe-source-han-serif-otc-fonts
base-devel
```

#### Configuring Nvidia

```sh
mhwd -a [pci or usb connection] [free or nonfree drivers] 0300 
```

#### Installing with `yay`

```sh
sudo pacman -Syyu yay; yay google-chrome

# https://www.tecmint.com/install-yay-aur-helper-in-arch-linux-and-manjaro/
yay -S wemeet-bin

# visual-studio code
pacman -S --needed base-devel
yay -S visual-studio-code-bin

git clone https://aur.archlinux.org/visual-studio-code-bin.git
cd visual-studio-code-bin; makepkg -i 
```

#### Installing with `pamac`

```sh
rm -f /var/lib/pacman/sync/ 
pamac update --force-refresh

pamac build visual-studio-code-bin
pamac build google-chrome
```

#### `npm` and `node`

```sh
sudo pacman -S nvm
echo 'source /usr/share/nvm/init-nvm.sh' >> ~/.zshrc
```

~~#### Installing `honkit`~~
Switched to ```gohugo```

```sh
nvm install --lts
npm install honkit-plugin-plantuml-server

vim book.json
{
  "plugins": [
    "plantuml-server"
  ]
}
```

#### `miniconda` Configuring

#### `plantuml` support in VScode

- install vsCode, then `markdown-preview-enhanced` extenstion
- In VScode > `Settings` > type `markdown-preview-enhanced: plantuml`
- Provide the exact path of `plantuml.jar`

#### `grub-rescue`

1. Rescue mode
> https://forum.manjaro.org/t/manjaro-wont-boot-after-windows-update/31109

```sh
ls (hd0,6)/boot
set root=(hd0,6)
set prefix=(hd0,6)/boot/grub
insmod normal
normal
```

2. Update grub permanently
> https://wiki.manjaro.org/index.php/GRUB/Restore_the_GRUB_Bootloader#Reinstall_GRUB

```sh
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=manjaro --recheck 
grub-mkconfig -o /boot/grub/grub.cfg
```