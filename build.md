sudo mkdir /cannon
sudo chown t:t /cannon
cp /mnt/D/DevTools/.ssh/* ~/.ssh/
chmod 700 ~/.ssh/*
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
export PATH=~/bin:$PATH
 git config --global user.email "tian@gtian.top"
 git config --global user.name "God"
repo init -u git@github.com:LineageOS/android.git -b lineage-20.0 --git-lfs --depth=1 -g default,-notdefault,-infra --no-clone-bundle
mkdir -p .repo/local_manifests
cat <<EOF > .repo/local_manifests/cannon.xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
    <remote name="devs"
            fetch="ssh://git@github.com"
            revision="lineage-20" />

    <project path="device/xiaomi/cannon"
             name="xiaomi-mt6853-devs/android_device_xiaomi_cannon"
             remote="devs" />

    <project path="vendor/xiaomi/cannon"
             name="xiaomi-mt6853-devs/android_vendor_xiaomi_cannon"
             remote="devs" />

    <project path="kernel/xiaomi/cannon"
             name="xiaomi-mt6853-devs/android_kernel_xiaomi_cannon"
             remote="devs" />
    <project path="device/mediatek/sepolicy_vndr"
             name="LineageOS/android_device_mediatek_sepolicy_vndr"
             remote="github"
             revision="lineage-20" />
</manifest>
EOF
repo sync -c --force-sync --force-remove-dirty --no-tags --no-clone-bundle  --jobs-checkout=$(nproc) --optimized-fetch --prune
cd /cannon
mkdir -p /cannon/android-certs && cd /cannon/android-certs
if [ ! -f releasekey.pk8 ]; then
    SUBJECT='/C=CN/ST=ShangHai/L=JingAn/O=GOD/OU=GOD/CN=Android'
    for key in releasekey platform shared media networkstack bluetooth sdk_sandbox; do \
        /cannon/development/tools/make_key $key "$SUBJECT"; \
    done
    openssl genrsa -out avb_release.pem 2048
fi
cd /cannon
sed -i 's|external/avb/test/data/testkey_rsa2048.pem|android-certs/avb_release.pem|g' device/xiaomi/cannon/BoardConfig.mk
sudo apt update && sudo apt install -y bc bison build-essential ccache curl flex g++-multilib gcc-multilib git git-lfs gnupg gperf imagemagick lib32readline-dev lib32z1-dev libelf-dev liblz4-tool libncurses5 libncurses5-dev libsdl1.2-dev libssl-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev git-lfs
sudo ln -sf /usr/bin/python3 /usr/bin/python
git lfs install
cd /cannon/external/chromium-webview/prebuilt/arm64
git lfs pull
cd /cannon/external/chromium-webview/prebuilt/arm
git lfs pull
cd /cannon/packages/modules/NetworkStack/res/values/
sed -i 's|http://connectivitycheck.gstatic.com/generate_204|http://connect.rom.miui.com/generate_204|g' config.xml
sed -i 's|https://www.google.com/generate_204|https://connect.rom.miui.com/generate_204|g' config.xml
sed -i 's|http://www.google.com/gen_204|http://connectivitycheck.platform.hicloud.com/generate_204|g' config.xml
sed -i 's|http://play.googleapis.com/generate_204|http://connectivitycheck.platform.hicloud.com/generate_204|g' config.xml
sed -i '/name="config_ntpServer"/s/>.*</>ntp.aliyun.com</' /cannon/frameworks/base/core/res/res/values/config.xml
sed -i '/name="config_ntpPollingInterval"/s/>.*</>12000000</' /cannon/frameworks/base/core/res/res/values/config.xml
sed -i '/name="config_ntpPollingIntervalShorter"/s/>.*</>6000</' /cannon/frameworks/base/core/res/res/values/config.xml
cd /cannon
CANNON_MK="device/xiaomi/cannon/lineage_cannon.mk"
echo "PRODUCT_DEFAULT_DEV_CERTIFICATE := android-certs/releasekey" >> $CANNON_MK
echo "PRODUCT_OTACERT := android-certs/releasekey.x509.pem" >> $CANNON_MK
sudo fallocate -l 80G /swapfile 
sudo chmod 600 /swapfile 
sudo mkswap /swapfile 
sudo swapon /swapfile
cd /cannon/
source build/envsetup.sh
export BUILD_NUMBER=20260101
export USER=GOD
lunch lineage_cannon-user
m -j$(nproc) bacon
# 生成签名的 Target Files (底包)
## 进入源码根目录
cd /cannon
## 运行签名工具
./build/tools/releasetools/sign_target_files_apks \
    -d android-certs/ \
    --extra_recovery_keys android-certs/releasekey \
    out/target/product/cannon/obj/PACKAGING/target_files_intermediates/lineage_cannon-target_files-*.zip \
    signed-target-files.zip
## 将签名的底包转为可刷写的 OTA 包	
./build/tools/releasetools/ota_from_target_files \
    -k android-certs/releasekey \
    signed-target-files.zip \
    lineage-20.0-cannon-signed-final.zip
