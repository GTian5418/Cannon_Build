# Xiaomi Cannon (Redmi Note 9T / 5G) 编译与签名自动化手册

## 一、 环境初始化与源码同步
```bash
# 1. 创建工作目录并授权
sudo mkdir -p /cannon
sudo chown $(whoami):$(whoami) /cannon
cd /cannon

# 2. SSH 密钥与工具配置
cp /mnt/D/DevTools/.ssh/* ~/.ssh/
chmod 700 ~/.ssh/*
mkdir -p ~/bin
curl [https://storage.googleapis.com/git-repo-downloads/repo](https://storage.googleapis.com/git-repo-downloads/repo) > ~/bin/repo
chmod a+x ~/bin/repo
export PATH=~/bin:$PATH

# 3. Git 全局配置
git config --global user.email "tian@gtian.top"
git config --global user.name "God"

# 4. 初始化 Repo 仓库 (Lineage-20.0)
repo init -u git@github.com:LineageOS/android.git -b lineage-20.0 \
    --git-lfs --depth=1 -g default,-notdefault,-infra --no-clone-bundle

# 5. 配置 Local Manifests (Xiaomi Cannon)
mkdir -p .repo/local_manifests
cat <<EOF > .repo/local_manifests/cannon.xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
    <remote name="devs" fetch="ssh://git@github.com" revision="lineage-20" />
    <project path="device/xiaomi/cannon" name="xiaomi-mt6853-devs/android_device_xiaomi_cannon" remote="devs" />
    <project path="vendor/xiaomi/cannon" name="xiaomi-mt6853-devs/android_vendor_xiaomi_cannon" remote="devs" />
    <project path="kernel/xiaomi/cannon" name="xiaomi-mt6853-devs/android_kernel_xiaomi_cannon" remote="devs" />
    <project path="device/mediatek/sepolicy_vndr" name="LineageOS/android_device_mediatek_sepolicy_vndr" remote="github" revision="lineage-20" />
</manifest>
EOF

# 6. 同步源码
repo sync -c --force-sync --force-remove-dirty --no-tags --no-clone-bundle --jobs-checkout=$(nproc) --optimized-fetch --prune
# 签名证书生成 (Release Keys)
mkdir -p /cannon/android-certs && cd /cannon/android-certs

# 仅在证书不存在时生成，防止覆盖
if [ ! -f releasekey.pk8 ]; then
    SUBJECT='/C=CN/ST=ShangHai/L=JingAn/O=GOD/OU=GOD/CN=Android'
    for key in releasekey platform shared media networkstack bluetooth sdk_sandbox; do \
        /cannon/development/tools/make_key $key "$SUBJECT"; \
    done
    openssl genrsa -out avb_release.pem 2048
fi

# 系统配置定制 (本地化优化)
cd /cannon

# 1. 替换 AVB 签名密钥
sed -i 's|external/avb/test/data/testkey_rsa2048.pem|android-certs/avb_release.pem|g' device/xiaomi/cannon/BoardConfig.mk

# 2. 依赖包安装与 Python 环境
sudo apt update && sudo apt install -y bc bison build-essential ccache curl flex g++-multilib gcc-multilib git git-lfs gnupg gperf imagemagick lib32readline-dev lib32z1-dev libelf-dev liblz4-tool libncurses5 libncurses5-dev libsdl1.2-dev libssl-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev
sudo ln -sf /usr/bin/python3 /usr/bin/python

# 3. WebView LFS 资源拉取
git lfs install
cd /cannon/external/chromium-webview/prebuilt/arm64 && git lfs pull
cd /cannon/external/chromium-webview/prebuilt/arm && git lfs pull
cd /cannon

# 4. 网络连接检测 (Captive Portal) 替换为国内地址
CP_FILE="packages/modules/NetworkStack/res/values/config.xml"
sed -i 's|[http://connectivitycheck.gstatic.com/generate_204](http://connectivitycheck.gstatic.com/generate_204)|[http://connect.rom.miui.com/generate_204](http://connect.rom.miui.com/generate_204)|g' $CP_FILE
sed -i 's|[https://www.google.com/generate_204](https://www.google.com/generate_204)|[https://connect.rom.miui.com/generate_204](https://connect.rom.miui.com/generate_204)|g' $CP_FILE
sed -i 's|[http://www.google.com/gen_204](http://www.google.com/gen_204)|[http://connectivitycheck.platform.hicloud.com/generate_204](http://connectivitycheck.platform.hicloud.com/generate_204)|g' $CP_FILE
sed -i 's|[http://play.googleapis.com/generate_204](http://play.googleapis.com/generate_204)|[http://connectivitycheck.platform.hicloud.com/generate_204](http://connectivitycheck.platform.hicloud.com/generate_204)|g' $CP_FILE

# 5. NTP 服务器优化 (阿里云)
NTP_FILE="frameworks/base/core/res/res/values/config.xml"
sed -i '/name="config_ntpServer"/s/>.*</>ntp.aliyun.com</' $NTP_FILE
sed -i '/name="config_ntpPollingInterval"/s/>.*</>12000000</' $NTP_FILE
sed -i '/name="config_ntpPollingIntervalShorter"/s/>.*</>6000</' $NTP_FILE

# 6. 强制指定 Recovery 签名证书 (解决 Error 21)
CANNON_MK="device/xiaomi/cannon/lineage_cannon.mk"
sed -i '/PRODUCT_DEFAULT_DEV_CERTIFICATE/d' $CANNON_MK
sed -i '/PRODUCT_OTACERT/d' $CANNON_MK
echo "PRODUCT_DEFAULT_DEV_CERTIFICATE := android-certs/releasekey" >> $CANNON_MK
echo "PRODUCT_OTACERT := android-certs/releasekey.x509.pem" >> $CANNON_MK
# 编译与最终签名
# 1. 开启 80G Swap 空间 (防止编译 OOM)
if [ ! -f /swapfile ]; then
    sudo fallocate -l 80G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
fi

# 2. 开始编译
source build/envsetup.sh
export BUILD_NUMBER=20260101
export USER=GOD
lunch lineage_cannon-user
m -j$(nproc) bacon

# 3. 签名 Target Files (核心步骤)
cd /cannon
./build/tools/releasetools/sign_target_files_apks \
    -d android-certs/ \
    --extra_recovery_keys android-certs/releasekey \
    out/target/product/cannon/obj/PACKAGING/target_files_intermediates/lineage_cannon-target_files-*.zip \
    signed-target-files.zip

# 4. 生成最终可刷写的签名 OTA 包
./build/tools/releasetools/ota_from_target_files \
    -k android-certs/releasekey \
    signed-target-files.zip \
    lineage-20.0-cannon-signed-final.zip
