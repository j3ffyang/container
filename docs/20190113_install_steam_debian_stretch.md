# How to Install Steam on Debian Stretch

I have an account at [Steam.com](https://store.steampowered.com/) and used to play some shooting games on Windows on one old laptop. Recently I decided to enable Steam on my Debian Stretch on my Dell XPS Developer Edition. It's running smoothly.

#### Edit ```/etc/apt/sources.list``` and Add

```
deb http://http.debian.net/debian/ stretch main contrib non-free
```

#### Enable multi-arch Support

```
sudo dpkg --add-architecture i386
```

Then

```
sudo apt update
```

#### Install Steam

```
sudo apt install steam
```

There might be some OpenGL 32-bit libraries installed.

#### Detect Video Chipset

```
lspci -v | grep -i --color vga

00:02.0 VGA compatible controller: Intel Corporation Device 5917 (rev 07) (prog-if 00 [VGA controller])
```

If you have Nvidia hardware,

```
sudo apt install libgl1-nvidia-glx:i386
```

or AMD graphics hardware,
```
sudo apt install libgl1-fglrx-glx:i386
```

Reference to https://wiki.debian.org/Steam
