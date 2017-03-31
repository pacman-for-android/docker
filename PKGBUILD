# Maintainer: Sébastien "Seblu" Luttringer

pkgname=docker
pkgver=17.03.1
pkgrel=1
epoch=1
pkgdesc='Pack, ship and run any application as a lightweight container'
arch=('x86_64' 'i686')
url='https://www.docker.com/'
license=('Apache')
depends=('bridge-utils' 'iproute2' 'device-mapper' 'sqlite' 'systemd' 'libseccomp')
makedepends=('git' 'go' 'btrfs-progs' 'go-md2man' 'cmake')
optdepends=('btrfs-progs: btrfs backend support'
            'lxc: lxc backend support')
# don't strip binaries! A sha1 is used to check binary consistency.
options=('!strip')
# Use exact commit version from Dockerfile for runc and containerd until 1.0.0
# https://github.com/docker/containerd/issues/299#issuecomment-240745119
# see commit in hack/dockerfile/binaries-commits
_RUNC_COMMIT=54296cf40ad8143b62dbcaa1d90e520a2136ddfe
_CONTAINERD_COMMIT=4ab9917febca54791c5f071a9d1f404867857fcc
_LIBNETWORK_COMMIT=0f534354b813003a754606689722fe253101bc4e
_TINI_COMMIT=949e6facb77383876aeff8a6944dde66b3089574
source=("git+https://github.com/docker/docker.git#tag=v$pkgver-ce"
        "git+https://github.com/docker/runc.git#commit=$_RUNC_COMMIT"
        "git+https://github.com/docker/containerd.git#commit=$_CONTAINERD_COMMIT"
        "git+https://github.com/docker/libnetwork.git#commit=$_LIBNETWORK_COMMIT"
        "git+https://github.com/krallin/tini.git#commit=$_TINI_COMMIT"
        "$pkgname.sysusers")
md5sums=('SKIP'
         'SKIP'
         'SKIP'
         'SKIP'
         'SKIP'
         '9a8b2744db23b14ca3cd350fdf73c179')

prepare() {
  cd docker
  # apply patch from the source array (should be a pacman feature)
  local filename
  for filename in "${source[@]}"; do
    if [[ "$filename" =~ \.patch$ ]]; then
      msg2 "Applying patch ${filename##*/}"
      patch -p1 -N -i "$srcdir/${filename##*/}"
    fi
  done
  :
}

build() {
  # check packager mistake on commit version
  msg2 'Checking commit mismatch'
  local _cfile _commit _pkgbuild _dockerfile
  _cfile="$srcdir"/docker/hack/dockerfile/binaries-commits
  . "$_cfile"
  for _commit in RUNC CONTAINERD LIBNETWORK TINI; do
    _pkgbuild=_${_commit}_COMMIT
    _dockerfile=${_commit}_COMMIT
    if [[ ${!_pkgbuild} != ${!_dockerfile} ]]; then
      error "Invalid $_commit commit"
      fgrep '_COMMIT=' "$_cfile"
      return 1
    fi
  done

  # go
  export GOPATH="$srcdir"

  # runc
  msg2 'Building runc'
  mkdir -p src/github.com/opencontainers
  ln -rsf "$srcdir/runc" src/github.com/opencontainers/runc
  cd src/github.com/opencontainers/runc
  make BUILDTAGS='seccomp'

  # containerd
  msg2 'Building containerd'
  cd "$srcdir"
  mkdir -p src/github.com/docker
  ln -rsf containerd src/github.com/docker
  cd src/github.com/docker/containerd
  LDFLAGS= make

  # docker proxy
  msg2 'Building docker-proxy'
  cd "$srcdir"
  ln -rsf libnetwork src/github.com/docker
  cd src/github.com/docker/libnetwork
  go build github.com/docker/libnetwork/cmd/proxy

  # docker-init
  msg2 'Building docker-init'
  cd "$srcdir/tini"
  cmake .
  # we must use the static binary because it's started in a foreign os
  make tini-static

  # docker
  msg2 'Building docker'
  cd "$srcdir"/docker
  export AUTO_GOPATH=1 DOCKER_BUILDTAGS='seccomp'
  hack/make.sh dynbinary
  # man pages
  man/md2man-all.sh 2>/dev/null
}

package() {
  # runc
  install -Dm755 runc/runc "$pkgdir/usr/bin/docker-runc"
  # docker-containerd
  install -Dm755 containerd/bin/containerd "$pkgdir/usr/bin/docker-containerd"
  install -Dm755 containerd/bin/containerd-shim "$pkgdir/usr/bin/docker-containerd-shim"
  install -Dm755 containerd/bin/ctr "$pkgdir/usr/bin/docker-containerd-ctr"
  # docker-proxy
  install -Dm755 libnetwork/proxy "$pkgdir/usr/bin/docker-proxy"
  # docker-init
  install -Dm755 tini/tini-static "$pkgdir/usr/bin/docker-init"
  # docker binary
  cd docker
  install -Dm755 "bundles/latest/dynbinary-client/docker" "$pkgdir/usr/bin/docker"
  install -Dm755 "bundles/latest/dynbinary-daemon/dockerd" "$pkgdir/usr/bin/dockerd"
  # completion
  install -Dm644 'contrib/completion/bash/docker' "$pkgdir/usr/share/bash-completion/completions/docker"
  install -Dm644 'contrib/completion/zsh/_docker' "$pkgdir/usr/share/zsh/site-functions/_docker"
  install -Dm644 'contrib/completion/fish/docker.fish' "$pkgdir/usr/share/fish/vendor_completions.d/docker.fish"
  # systemd
  install -Dm644 'contrib/init/systemd/docker.service' "$pkgdir/usr/lib/systemd/system/docker.service"
  install -Dm644 'contrib/init/systemd/docker.socket' "$pkgdir/usr/lib/systemd/system/docker.socket"
  install -Dm644 'contrib/udev/80-docker.rules' "$pkgdir/usr/lib/udev/rules.d/80-docker.rules"
  install -Dm644 "$srcdir/$pkgname.sysusers" "$pkgdir/usr/lib/sysusers.d/$pkgname.conf"
  # vim syntax
  install -Dm644 'contrib/syntax/vim/syntax/dockerfile.vim' "$pkgdir/usr/share/vim/vimfiles/syntax/dockerfile.vim"
  install -Dm644 'contrib/syntax/vim/ftdetect/dockerfile.vim' "$pkgdir/usr/share/vim/vimfiles/ftdetect/dockerfile.vim"
  # man
  install -dm755 "$pkgdir/usr/share/man"
  mv man/man* "$pkgdir/usr/share/man"
}

# vim:set ts=2 sw=2 et:
