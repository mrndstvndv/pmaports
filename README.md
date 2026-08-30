# Personal pmaports recipes

Personal postmarketOS APKBUILD recipes and local package overlays.

This repository is an overlay collection rather than a replacement for the
upstream postmarketOS `pmaports` tree. Copy the recipe directories into the
pmaports checkout used by pmbootstrap, or use this tree as the local recipe
source when the required base packages are available from Alpine/postmarketOS
repositories.

The current collection contains:
- Hyprland ecosystem recipes and the tapviz plugin (`main/`)
- Xiaomi POCO X3 Pro (`vayu`) device configs and firmware (`device/testing/`)
- Mainline SM8150 Linux kernel package with UFS auto-hibern8 stability fix (`device/testing/linux-postmarketos-qcom-sm8150/`)

