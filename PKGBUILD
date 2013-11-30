# Maintainer: Sébastien "Seblu" Luttringer

pkgname=docker
pkgver=0.7.0
pkgrel=1
epoch=1
pkgdesc='Pack, ship and run any application as a lightweight container'
arch=('x86_64' 'i686')
url='http://www.docker.io/'
license=('Apache')
depends=('bridge-utils' 'iproute2' 'device-mapper' 'lxc' 'sqlite' 'systemd')
makedepends=('git' 'go')
# don't strip binaries! A sha1 is used to check binary consistency.
options=('!strip')
source=("git+https://github.com/dotcloud/docker.git#tag=v$pkgver"
        'docker.service'
        '01-fix-libexec.patch')
md5sums=('SKIP'
         '3f7ccab915fb1942f06e18946c2811d2'
         '6684c04dd4697e0cb7aa3f171c96a2e4')
# magic harcoded path
_magic=src/github.com/dotcloud

prepare() {
  mkdir -p "$_magic"
  ln -sfn "../../../docker" "$_magic/docker"
  # patch for libexec path waiting 0.7.1
  patch -p1 -d docker < 01-fix-libexec.patch
}

build() {
  cd "$_magic/docker"
  export GOPATH="$srcdir:$srcdir/$_magic/docker/vendor"
  ./hack/make.sh dynbinary
}

check() {
  cd "$_magic/docker"
  # Will be added upstream soon
  #./hack/make.sh dyntest
}

package() {
  cd "$_magic/docker"
  install -Dm755 "bundles/$pkgver/dynbinary/docker-$pkgver" "$pkgdir/usr/bin/docker"
  install -Dm755 "bundles/$pkgver/dynbinary/dockerinit-$pkgver" "$pkgdir/usr/lib/docker/dockerinit"
  # completion
  install -Dm644 "contrib/completion/bash/docker" "$pkgdir/usr/share/bash-completion/completions/docker"
  install -Dm644 "contrib/completion/zsh/_docker" "$pkgdir/usr/share/zsh/site-functions/_docker"
  # systemd
  install -Dm644 "$srcdir/docker.service" "$pkgdir/usr/lib/systemd/system/docker.service"
}

# vim:set ts=2 sw=2 et:
