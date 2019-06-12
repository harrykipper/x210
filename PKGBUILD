# Maintainer: Jan de Groot <jgc@archlinux.org>

pkgname=upower
pkgver=0.99.10
pkgrel=3
pkgdesc="Abstraction for enumerating power devices, listening to device events and querying history and statistics"
url="https://upower.freedesktop.org"
arch=(x86_64)
license=(GPL)
depends=(systemd libusb libimobiledevice libgudev)
makedepends=(intltool docbook-xsl gobject-introspection python git gtk-doc)
backup=(etc/UPower/UPower.conf)
_commit=215049e7b80c5f24cb35cd229a445c6cf19bd381  # tags/UPOWER_0_99_10^0
source=("git+https://gitlab.freedesktop.org/upower/upower.git#commit=$_commit")
md5sums=('SKIP')

pkgver() {
  cd $pkgname
  git describe --tags | sed -e 's/UPOWER_//' -e 's/_/\./g' -e 's/-/+/g'
}

prepare() {
  cd $pkgname
  patch -p0 < ../../x210.patch
  NOCONFIGURE=1 ./autogen.sh
}

build() {
  cd $pkgname
  ./configure \
    --prefix=/usr \
    --sysconfdir=/etc \
    --localstatedir=/var \
    --libexecdir=/usr/lib \
    --disable-static \
    --enable-gtk-doc
  make
}

package() {
  cd $pkgname
  make DESTDIR="$pkgdir" install
}
