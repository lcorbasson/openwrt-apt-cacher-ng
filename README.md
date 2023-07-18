# OpenWrt apt-cacher-ng feed

## News
* 2020-06-08 Update to openwrt 19.07.3- apt-cacher-ng 3.6.3
* 2022-11-27 Update to apt-cacher-ng 3.6.4

## Build the package from apt-cacher-ng feed
1. Create a local development directory
    ```
    mkdir -p "$HOME/devel/openwrt"
    cd "$HOME/devel/openwrt"
    ```

1. Get information on your router's board and OpenWRT release and set useful variables:
    ```
    ssh root@router "cat /etc/openwrt_release"
    eval "$(ssh root@router "cat /etc/openwrt_release")"
    export APT_CACHER_NG_VERSION="3.6.4-1"
    ```

1. Download [the OpenWRT SDK](https://openwrt.org/docs/guide-developer/toolchain/using_the_sdk). You have to select and download the SDK for your particular board. As an example, you might get:
    ```
    export OPENWRT_SDK_ABI="gcc-8.4.0_musl_eabi"
    wget "https://downloads.openwrt.org/releases/${DISTRIB_RELEASE}/targets/${DISTRIB_TARGET}/openwrt-sdk-${DISTRIB_RELEASE}-${DISTRIB_TARGET/\//-}_${OPENWRT_SDK_ABI}.$(uname -s)-$(uname -p).tar.xz"
    ```

1. Extract the SDK:
    ```
    tar -xJf "openwrt-sdk-${DISTRIB_RELEASE}-${DISTRIB_TARGET/\//-}_${OPENWRT_SDK_ABI}.$(uname -s)-$(uname -p).tar.xz"
    ```

1. Use the provided feed:
    * a. With local modifications enabled:
        - Download the `apt-cacher-ng` feed from GitHub:
            ```
            git clone "https://github.com/vasvir/openwrt-packages.git" "apt-cacher-ng"
            ```
        - Edit `feeds.conf.default` in the SDK downloaded and extracted before. Add the following line, replacing `$HOME/devel/openwrt/apt-cacher-ng` by the absolute path to your local directory (e.g. `/home/bill/devel/openwrt/apt-cacher-ng`:
            ```
            src-link aptcacherng $HOME/devel/openwrt/apt-cacher-ng
            ```

    * b. Or directly from GitHub:
        - Adjust `feeds.conf.default` by adding:
            ```
            src-git aptcacherng https://github.com/vasvir/openwrt-packages.git
            ```

1. Configure local packages:
    ```
    cd "openwrt-sdk-${DISTRIB_RELEASE}-${DISTRIB_TARGET/\//-}_${OPENWRT_SDK_ABI}.$(uname -s)-$(uname -p)"
    ./scripts/feeds update -a
    ./scripts/feeds install apt-cacher-ng
    make menuconfig
    ```

    `menuconfig` should show `apt-cacher-ng` as a module (`<M>`) under `Network/Web Servers/Proxies` [1]. Save and exit.

1. Install local signing keys:
    ```
    ./staging_dir/host/bin/usign -G -s ./key-build -p ./key-build.pub -c "Local build key"
    ```

1. Build the package [2]:
    ```
    make -j
    ```

1. Copy the IPK file over to the OpenWRT router:
    ```
    scp "bin/packages/${DISTRIB_ARCH}/aptcacherng/apt-cacher-ng_${APT_CACHER_NG_VERSION}_${DISTRIB_ARCH}.ipk" root@router:
    ```

1. Install the apt-cacher-ng package [3]:
    ```
    ssh root@router opkg update
    ssh root@router opkg install "apt-cacher-ng_${APT_CACHER_NG_VERSION}_${DISTRIB_ARCH}.ipk"
    ```
    
1. Configure the apt-cacher-ng service
    ```
    ssh root@router
    ```
    
    * Edit `/etc/apt-cacher-ng/acng.conf` to configure the `CacheDir` and `LogDir`. You'll probably want to [use USB storage](https://openwrt.org/docs/guide-user/storage/usb-drives-quickstart) for this, e.g. here `/mnt/sda1`:
    ```
    # Storage directory for downloaded data and related maintenance activity.
    #
    # Note: When the value for CacheDir is changed, change the file
    # /lib/systemd/system/apt-cacher-ng.service too
    #
    #CacheDir: /var/tmp
    CacheDir: /mnt/sda1/apt-cacher-ng/cache
    
    # Log file directory, can be set empty to disable logging
    #
    #LogDir: /var/tmp
    LogDir: /mnt/sda1/apt-cacher-ng/log
    ```
    * Create the `CacheDir`, `LogDir` and `/var/run/apt-cacher-ng` directories, and change their ownership to `apt-cacher-ng.apt-cacher.ng`:
    ```
    mkdir -p "/mnt/sda1/apt-cacher-ng/cache" "/mnt/sda1/apt-cacher-ng/log" "/var/run/apt-cacher-ng"
    chown apt-cacher-ng.apt-cacher.ng "/mnt/sda1/apt-cacher-ng/cache" "/mnt/sda1/apt-cacher-ng/log" "/var/run/apt-cacher-ng"
    ```

1. Enable and restart the service
    ```
    /etc/init.d/apt-cacher-ng enable
    /etc/init.d/apt-cacher-ng restart
    ```

1. Configure the clients, e.g. by following [these instructions for Debian](https://wiki.debian.org/AptCacherNg) and its derivatives like Ubuntu.


## TODO
* Autoconfigure step 11

## Troubleshooting
This is possible only if you have followed step 3a.

friendly commands
* make V=s
* make V=s package/feeds/local/apt-cacher-ng/compile
* make V=s package/feeds/local/apt-cacher-ng/install

### [1] Buildroot Makefile problem.
Adjust /home/bill/Downloads/hardware/linksys1200ac/openwrt-packages/apt-cacher-ng/Makefile and retry
```
./scripts/feeds uninstall apt-cacher-ng
rm -rf "build_dir/target-${DISTRIB_ARCH}_musl_eabi/apt-cacher-ng-${APT_CACHER_NG_VERSION%-*}/"
./scripts/feeds install apt-cacher-ng
make menuconfig
```

This may help if you need to update dependencies
```
./scripts/feeds update -a
```

### [2] Package CMakeLists.txt problem.
Adjust /home/bill/Downloads/hardware/linksys1200ac/openwrt-packages/apt-cacher-ng/patches/000-add_install_target.patch and retry:
* setup once:
    * download the apt-cacher-ng source
        ```
        wget "http://ftp.us.debian.org/debian/pool/main/a/apt-cacher-ng/apt-cacher-ng_${APT_CACHER_NG_VERSION%-*}.orig.tar.xz"
        ```

    * extract it
         ```
         tar -xJf "apt-cacher-ng_${APT_CACHER_NG_VERSION%-*}.orig.tar.xz"
         ```

    * rename and copy it to have an easy diff target
         ```
         mv "apt-cacher-ng_${APT_CACHER_NG_VERSION%-*}" a
         cp -a a b
         ```

* edit test cycle
    * modify b - produce patches for CMakeLists.txt and move them over to /home/bill/Downloads/hardware/linksys1200ac/openwrt-packages/apt-cacher-ng/patches/000-add_install_target.patch
        ```
       diff -ur a b > /home/bill/Downloads/hardware/linksys1200ac/openwrt-packages/apt-cacher-ng/patches/000-add_install_target.patch
        ```

    * build
        ```
        rm -rf "build_dir/target-${DISTRIB_ARCH}_musl_eabi/apt-cacher-ng-${APT_CACHER_NG_VERSION%-*}/"
        make V=s
        ```

### [3] Installation problems in postinst.
Adjust Buildroot Makefile and retry:
```
ssh root@openwrt opkg remove apt-cacher-ng
./scripts/feeds uninstall apt-cacher-ng
rm -rf "build_dir/target-${DISTRIB_ARCH}_musl_eabi/apt-cacher-ng-${APT_CACHER_NG_VERSION%-*}/"
./scripts/feeds install apt-cacher-ng
make -j
scp "bin/packages/${DISTRIB_ARCH}/local/apt-cacher-ng_${APT_CACHER_NG_VERSION}_${DISTRIB_ARCH}.ipk" root@openwrt:
ssh root@openwrt opkg install "apt-cacher-ng_${APT_CACHER_NG_VERSION}_${DISTRIB_ARCH}.ipk"
```
