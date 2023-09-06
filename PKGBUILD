# Maintainer: Sébastien "Seblu" Luttringer
# Maintainer: Morten Linderud <foxboron@archlinux.org>

pkgname=docker
pkgver=24.0.5
pkgrel=1.1
epoch=1
pkgdesc='Pack, ship and run any application as a lightweight container'
arch=('x86_64' 'aarch64')
url='https://www.docker.com/'
license=('Apache')
depends=('glibc' 'bridge-utils' 'iproute2' 'device-mapper' 'sqlite' 'systemd-libs'
         'libseccomp' 'libtool' 'runc' 'containerd')
makedepends=('git' 'go' 'btrfs-progs' 'cmake' 'systemd' 'go-md2man' 'sed')
optdepends=('btrfs-progs: btrfs backend support'
            'pigz: parallel gzip compressor support'
            'docker-scan: vulnerability scanner'
            'docker-buildx: extended build capabilities')
options=(!lto)
# https://github.com/moby/moby/tree/v20.10.0/hack/dockerfile/install
_TINI_COMMIT=de40ad007797e0dcd8b7126f27bb87401d224240
source=("git+https://github.com/docker/cli.git#tag=v$pkgver"
        "git+https://github.com/moby/moby.git#tag=v$pkgver"
        "git+https://github.com/krallin/tini.git#commit=$_TINI_COMMIT"
        "$pkgname.sysusers"
        cli-change-prefix.patch
        moby-change-prefix.patch
        fix-tmp-for-man-page-gen.patch)
sha256sums=('SKIP'
            'SKIP'
            'SKIP'
            '541826011a9836d05a2f42293d5f1beadf2ca8d89fb604487d61a013505678eb'
            'bd04e80858dd48ab46cfe92f962428996f240d6f18d2288c2f76673dcc980eb9'
            '818a76fd7ebb2ab74456a636bab874d27dd52ba63d739ad0db089e0b0350b329'
            'dd4777e428e15aeaa772ec2b93bb2bc72bbda7fb82cd6416f410825dd7f5b0b1')

# create a fake go path directory and pushd into it
# $1 real directory
# $2 gopath directory
_fake_gopath_pushd() {
  mkdir -p "$GOPATH/src/${2%/*}"
  rm -f "$GOPATH/src/$2"
  ln -rsT "$1" "$GOPATH/src/$2"
  pushd  "$GOPATH/src/$2" >/dev/null
}

_fake_gopath_popd() {
  popd >/dev/null
}

prepare() {
  patch -Np1 -d cli < cli-change-prefix.patch
  patch -Np1 -d cli < fix-tmp-for-man-page-gen.patch
  patch -Np1 -d moby < moby-change-prefix.patch
  sed -i '1s|.*|#!/data/usr/bin/bash|' cli/scripts/{warn-outside-container,vendor,build/{.variables,binary,mkversioninfo,plugins},docs/*.sh} moby/hack/*.sh
}

build() {
  ### check my mistakes on commit version
  echo 'Checking commit mismatch'
  (
  local _cfile
  for _cfile in tini; do
    . "moby/hack/dockerfile/install/$_cfile.installer"
  done
  local _commit _pkgbuild _dockerfile
  err=0
  # FIXME: Do not check TINI anymore, use tag instead of commit
  # TODO: libnetwork is removed
  # for _commit in LIBNETWORK; do
  #   _pkgbuild=_${_commit}_COMMIT
  #   _dockerfile=${_commit}_COMMIT
  #   if [[ ${!_pkgbuild} != ${!_dockerfile} ]]; then
  #     echo "Invalid $_commit commit, should be ${!_dockerfile}" >&2
  #     err=$(($err + 1))
  #   fi
  # done
  return $err
  )

  ### globals
  export GOPATH="$srcdir"
  export PATH="$GOPATH/bin:$PATH"
  export CGO_CPPFLAGS="${CPPFLAGS}"
  export CGO_CFLAGS="${CFLAGS}"
  export CGO_CXXFLAGS="${CXXFLAGS}"
  export CGO_LDFLAGS="${LDFLAGS}"
  export LDFLAGS=''
  export GOFLAGS='-buildmode=pie -trimpath -mod=readonly -modcacherw -ldflags=-linkmode=external'
  export GO111MODULE=off
  export DISABLE_WARN_OUTSIDE_CONTAINER=1

  ### cli
  echo 'Building cli'
  _fake_gopath_pushd cli github.com/docker/cli
  make VERSION=$pkgver dynbinary
  make manpages
  _fake_gopath_popd

  ### daemon
  echo 'Building daemon'
  _fake_gopath_pushd moby github.com/docker/docker
  DOCKER_GITCOMMIT=$(cd "$srcdir"/moby && git rev-parse --short HEAD) \
    DOCKER_BUILDTAGS='seccomp apparmor' \
    VERSION=$pkgver \
    hack/make.sh dynbinary
  _fake_gopath_popd

  ### docker-init
  echo 'Building docker-init'
  _fake_gopath_pushd tini github.com/krallin/tini
  cmake .
  # we must use the static binary because it's started in a foreign os
  make tini-static
  _fake_gopath_popd
}

package() {
  ### init
  install -Dm755 tini/tini-static "$pkgdir/data/usr/bin/docker-init"
  ### dockerd
  install -Dm755 moby/bundles/dynbinary-daemon/dockerd "$pkgdir"/data/usr/bin/dockerd
  install -Dm755 moby/bundles/dynbinary-daemon/docker-proxy "$pkgdir/data/usr/bin/docker-proxy"
  ### systemd units
  cd "$srcdir"/moby/contrib
  install -Dm644 'init/systemd/docker.service' "$pkgdir/data/usr/lib/systemd/system/docker.service"
  install -Dm644 'init/systemd/docker.socket' "$pkgdir/data/usr/lib/systemd/system/docker.socket"
  # systemd rules
  install -Dm644 'udev/80-docker.rules' "$pkgdir/data/usr/lib/udev/rules.d/80-docker.rules"
  install -Dm644 "$srcdir/$pkgname.sysusers" "$pkgdir/data/usr/lib/sysusers.d/$pkgname.conf"
  ### cli
  cd "$srcdir"/cli
  # binary
  install -Dm755 build/docker "$pkgdir/data/usr/bin/docker"
  # completion (see FS#79067)
  install -Dm644 <(build/docker completion bash) "$pkgdir/data/usr/share/bash-completion/completions/docker"
  install -Dm644 <(build/docker completion zsh) "$pkgdir/data/usr/share/zsh/site-functions/_docker"
  install -Dm644 <(build/docker completion fish) "$pkgdir/data/usr/share/fish/vendor_completions.d/docker.fish"
  # man
  install -dm755 "$pkgdir/data/usr/share/man"
  cp -r man/man* "$pkgdir/data/usr/share/man"
}

# vim:set ts=2 sw=2 et:
