# Maintainer: Sébastien "Seblu" Luttringer

pkgname=docker
pkgver=17.05.0
pkgrel=1
epoch=1
pkgdesc='Pack, ship and run any application as a lightweight container'
arch=('x86_64' 'i686')
url='https://www.docker.com/'
license=('Apache')
depends=('glibc' 'bridge-utils' 'iproute2' 'device-mapper' 'sqlite' 'libsystemd'
         'libseccomp')
makedepends=('git' 'go' 'btrfs-progs' 'cmake')
optdepends=('btrfs-progs: btrfs backend support'
            'lxc: lxc backend support')
# don't strip binaries! A sha1 is used to check binary consistency.
options=('!strip')
# Use exact commit version from Dockerfile for runc and containerd until 1.0.0
# https://github.com/docker/containerd/issues/299#issuecomment-240745119
# see commit in hack/dockerfile/binaries-commits
_RUNC_COMMIT=9c2d8d184e5da67c95d601382adf14862e4f2228
_CONTAINERD_COMMIT=9048e5e50717ea4497b757314bad98ea3763c145
_TINI_COMMIT=949e6facb77383876aeff8a6944dde66b3089574
_LIBNETWORK_COMMIT=7b2b1feb1de4817d522cc372af149ff48d25028e
source=("git+https://github.com/moby/moby.git#tag=v$pkgver-ce"
        "git+https://github.com/docker/runc.git#commit=$_RUNC_COMMIT"
        "git+https://github.com/containerd/containerd.git#commit=$_CONTAINERD_COMMIT"
        "git+https://github.com/docker/libnetwork.git#commit=$_LIBNETWORK_COMMIT"
        "git+https://github.com/krallin/tini.git#commit=$_TINI_COMMIT"
        "git+https://github.com/spf13/cobra.git"
        "git+https://github.com/cpuguy83/go-md2man.git"
        "$pkgname.sysusers")
md5sums=('SKIP'
         'SKIP'
         'SKIP'
         'SKIP'
         'SKIP'
         'SKIP'
         'SKIP'
         '9a8b2744db23b14ca3cd350fdf73c179')

build() {
  ### check my mistakes on commit version
  msg2 'Checking commit mismatch'
  local _cfile _commit _pkgbuild _dockerfile
  _cfile="$srcdir"/moby/hack/dockerfile/binaries-commits
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

  ### go magics
  export GOPATH="$srcdir"
  export PATH="$GOPATH/bin:$PATH"

  ### docker binary
  msg2 'Building docker'
  # projects still reference import with github.com/docker
  mkdir -p src/github.com/docker
  ln -rsfT moby src/github.com/docker/docker
  pushd src/github.com/docker/docker >/dev/null
  DOCKER_BUILDTAGS='seccomp' hack/make.sh dynbinary
  popd >/dev/null

  ### go-md2man (used for manpages)
  msg2 'Building go-md2man'
  mkdir -p src/github.com/cpuguy83
  ln -rsf go-md2man src/github.com/cpuguy83
  pushd src/github.com/cpuguy83/go-md2man >/dev/null
  go get -v ./...
  popd >/dev/null

  ### docker man pages
  msg2 'Building man pages'
  # cobra (used for manpages)
  mkdir -p src/github.com/spf13
  ln -rsf cobra src/github.com/spf13
  # generate
  pushd src/github.com/docker/docker >/dev/null
  man/generate.sh 2>/dev/null
  popd >/dev/null

  ### runc
  msg2 'Building runc'
  pushd runc >/dev/null
  make BUILDTAGS='seccomp'
  popd >/dev/null

  ### containerd
  msg2 'Building containerd'
  ln -rsfT containerd src/github.com/docker/containerd
  pushd src/github.com/docker/containerd >/dev/null
  LDFLAGS= make
  popd >/dev/null

  ### docker proxy
  msg2 'Building docker-proxy'
  ln -rsf libnetwork src/github.com/docker
  pushd src/github.com/docker/libnetwork >/dev/null
  go build -ldflags='-linkmode=external' github.com/docker/libnetwork/cmd/proxy
  popd >/dev/null

  ### docker-init
  msg2 'Building docker-init'
  pushd tini >/dev/null
  cmake .
  # we must use the static binary because it's started in a foreign os
  make tini-static
  popd >/dev/null
}

package() {
  ### runc
  install -Dm755 runc/runc "$pkgdir/usr/bin/docker-runc"
  ### docker-containerd
  install -Dm755 containerd/bin/containerd "$pkgdir/usr/bin/docker-containerd"
  install -Dm755 containerd/bin/containerd-shim \
    "$pkgdir/usr/bin/docker-containerd-shim"
  install -Dm755 containerd/bin/ctr "$pkgdir/usr/bin/docker-containerd-ctr"
  ### docker-proxy
  install -Dm755 libnetwork/proxy "$pkgdir/usr/bin/docker-proxy"
  ### docker-init
  install -Dm755 tini/tini-static "$pkgdir/usr/bin/docker-init"
  ### docker binary
  cd moby
  install -Dm755 "bundles/latest/dynbinary-client/docker" \
    "$pkgdir/usr/bin/docker"
  install -Dm755 "bundles/latest/dynbinary-daemon/dockerd" \
    "$pkgdir/usr/bin/dockerd"
  ### completion
  install -Dm644 'contrib/completion/bash/docker' \
    "$pkgdir/usr/share/bash-completion/completions/docker"
  install -Dm644 'contrib/completion/zsh/_docker' \
    "$pkgdir/usr/share/zsh/site-functions/_docker"
  install -Dm644 'contrib/completion/fish/docker.fish' \
    "$pkgdir/usr/share/fish/vendor_completions.d/docker.fish"
  ### systemd
  install -Dm644 'contrib/init/systemd/docker.service' \
    "$pkgdir/usr/lib/systemd/system/docker.service"
  install -Dm644 'contrib/init/systemd/docker.socket' \
    "$pkgdir/usr/lib/systemd/system/docker.socket"
  install -Dm644 'contrib/udev/80-docker.rules' \
    "$pkgdir/usr/lib/udev/rules.d/80-docker.rules"
  install -Dm644 "$srcdir/$pkgname.sysusers" \
    "$pkgdir/usr/lib/sysusers.d/$pkgname.conf"
  ### vim syntax
  install -Dm644 'contrib/syntax/vim/syntax/dockerfile.vim' \
    "$pkgdir/usr/share/vim/vimfiles/syntax/dockerfile.vim"
  install -Dm644 'contrib/syntax/vim/ftdetect/dockerfile.vim' \
    "$pkgdir/usr/share/vim/vimfiles/ftdetect/dockerfile.vim"
  ### man
  install -dm755 "$pkgdir/usr/share/man"
  cp -r man/man* "$pkgdir/usr/share/man"
}

# vim:set ts=2 sw=2 et:
