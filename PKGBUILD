# AArch64 multi-platform
# Maintainer: Mahmut Dikcizgi <boogiepop a~t gmx com>
# Contributor: Kevin Mihelich <kevin@archlinuxarm.org>

_pkgver=6.5
_kernel_tag="panthor"
pkgbase=linux-$_kernel_tag-git
pkgname=("${pkgbase}-headers" $pkgbase)
pkgver=6.7.0.rc1.panthor.g3255db267a2b
pkgrel=1
arch=('aarch64')
license=('GPL2')
url="https://github.com/hbiyik"
pkgdesc="Linux kernel package targeting to pretest 6.5x merges for panthor" 
makedepends=('xmlto' 'docbook-xsl' 'kmod' 'inetutils' 'bc' 'git' 'uboot-tools' 'vboot-utils' 'dtc')
options=('!strip')

source=(git+$url/linux.git#branch=panthor+mpp+rga
        'linux.preset'
        "extlinux.${_kernel_tag}.template"
        )

b2sums=('SKIP'
        'bd296f775df973c6dcb6bd8311ce4d3af9a8d4a67905f17c450cae776aab0229987d473334d38fd102a34ed483a121f67ac58a48fd9e6fab2c714c7079e06613'
        '60ef0a5252d57212b79158c42928d255f62dd8139a57bb66e013f101aa78a54976850edb30d07b72dfbd52cff5e739176c59172dbededf53c3499fb7a72d263b')

pkgver(){
  #gets the commit count of both repos + _pkgrel and sums them to calculate the revision number
  cd linux
  rm localversion.*
  echo "-$_kernel_tag" > localversion.20-pkgname
  _version=$(make kernelrelease) 
  printf "${_version//-/\.}"
}

prepare() {
  cd linux

  echo "-$pkgrel" > localversion.10-pkgrel
  echo "-$_kernel_tag" > localversion.20-pkgname
  
  # this is only for local builds so there is no need to integrity check
  for p in ../../custom/*.patch; do
    echo "Custom Patching with ${p}"
    patch -p1 -N -i $p || true
  done

  if [ -f ../../custom/config ]; then
    echo "Using User Specific Config"
    cp -f ../../custom/config ./.config
  else 
    cp -f arch/arm64/configs/rk3588_panthor_release_defconfig ./.config
  fi
  
  scripts/kconfig/merge_config.sh -m .config arch/arm64/configs/mpp_defconfig
  scripts/kconfig/merge_config.sh -m .config arch/arm64/configs/rga_multi_defconfig
  make olddefconfig prepare
  make -s kernelrelease > version
}

build() {
  cd linux

  unset LDFLAGS
  make ${MAKEFLAGS} Image modules
  make ${MAKEFLAGS} DTC_FLAGS="-@" dtbs
}

_package-git() {
  pkgdesc="Linux kernel package targeting to pretest 6.5x merges for panthor"
  depends=('coreutils' 'kmod' 'mkinitcpio>=0.7' 'mali-valhall-g610-firmware')
  optdepends=('wireless-regdb: to set the correct wireless channels of your country')
  provides=("linux=${pkgver}" "linux-${_kernel_tag}")
  conflicts=("linux-${_kernel_tag}")
  backup=("etc/mkinitcpio.d/linux-${_kernel_tag}.preset")

  cd linux
  
  local _version="$(<version)"
  
  # install dtbs
  make INSTALL_DTBS_PATH="${pkgdir}/boot/dtbs/linux-$_kernel_tag" dtbs_install

  # install extlinux template
  install -Dm644 "../extlinux.${_kernel_tag}.template" "$pkgdir/boot/extlinux/extlinux.${_kernel_tag}.template"
  
  # install modules
  make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 modules_install

  # copy kernel
  install -Dm644 arch/arm64/boot/Image "$pkgdir/usr/lib/modules/$_version/vmlinuz"

  # sed expression for following substitutions
  local _subst="
    s|%PKGBASE%|linux-${_kernel_tag}|g
    s|%KERNVER%|${_version}|g
  "

  # used by mkinitcpio to name the kernel
  echo "linux-$_kernel_tag" | install -Dm644 /dev/stdin "$pkgdir/usr/lib/modules/$_version/pkgbase"

  # install mkinitcpio preset file
  sed "$_subst" ../linux.preset |
    install -Dm644 /dev/stdin "$pkgdir/etc/mkinitcpio.d/linux-$_kernel_tag.preset"
}

_package-git-headers() {
  pkgdesc="Linux kernel header package targeting to pretest 6.5x merges for panthor"
  depends=("python")
  provides=("linux-headers=${pkgver}" "linux-${_kernel_tag}-headers")
  conflicts=("linux-${_kernel_tag}-headers")

  cd linux
  local _version="$(<version)"
  local builddir="$pkgdir/usr/lib/modules/$_version/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map version
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/arm64" -m644 arch/arm64/Makefile
  cp -t "$builddir" -a scripts

  # add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/arm64" -a arch/arm64/include
  install -Dt "$builddir/arch/arm64/kernel" -m644 arch/arm64/kernel/asm-offsets.s
  mkdir -p "$builddir/arch/arm"
  cp -t "$builddir/arch/arm" -a arch/arm/include

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */arm64/ || $arch == */arm/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -bi "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name -print0)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/linux-$_kernel_tag"
}

for _p in ${pkgname[@]}; do
  eval "package_${_p}() {
    _package${_p#linux-${_kernel_tag}}
  }"
done