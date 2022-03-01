# Cleanly Manage Packages in Fedora on MacBookPro

#### Background

Recently I installed Fedora 35 on a MacBookPro 16,2 and want to specifically manage the update and installed packages with this customized kernel 

![20220224_neofetch_mbp-fedora](/assets/20220224_neofetch_mbp-fedora.png)

## Commands

#### Installed

- Installed packages

```sh
sudo dnf list installed | wc
```

- Check if `iwl*` installed

```sh
# All iwl* firmware is unnecessary on MacBookPro
sudo dnf list installed | grep iwl
```

#### Upgrade and Update

- List available upgrade

```sh
sudo dnf list upgrades
```

- Update a particular package

```sh
sudo dnf update `sudo dnf list upgrades | grep shadow-utils | awk '{print $1}'`
```

#### Remove

- Remove the unwanted package

```sh
sudo dnf remove `sudo dnf list installed | grep iwl | awk '{print $1}'`
```

#### Troubleshooting

- Check whether the firmware is loaded properly

```sh
[jeff@mbp ~]$ sudo dmesg | egrep -i 'error|critical|warn|failed'
[20892.988681] apple-ib-touchbar 0003:05AC:8302.000C: tb: Failed to set touch bar mode to 1 (-110)
[20895.036798] apple-ib-touchbar 0003:05AC:8302.000C: tb: Failed to set touch bar mode to 2 (-110)
[20897.085721] apple-ib-touchbar 0003:05AC:8302.000C: tb: Failed to set touch bar mode to 2 (-110)
[20899.132977] apple-ib-touchbar 0003:05AC:8302.000C: tb: Failed to set touch bar mode to 2 (-110)
```
