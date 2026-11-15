# Maintainer: MicLeh <micleh at proton dot me>

pkgname=karere-git
pkgver=4.5.0.r1.g988485d
pkgrel=1
pkgdesc="A fast, native WhatsApp client for Linux with GTK4/LibAdwaita (v4 CEF build)"
url="https://github.com/tobagin/karere"
arch=('x86_64' 'aarch64')
license=('GPL-3.0-or-later')
depends=(
  'gtk4'
  'libadwaita'
  'glib2'
  'dbus'
  'nss'
  'nspr'
  'alsa-lib'
  'libpulse'
  'fontconfig'
  'expat'
  'libxkbcommon'
  'libx11'
  'libxcb'
  'libxcomposite'
  'libxdamage'
  'libxext'
  'libxfixes'
  'libxrandr'
  'libxcursor'
  'libxi'
  'libxss'
  'libxinerama'
  'mesa'
  'libglvnd'
  'systemd-libs'
  'at-spi2-core'
)
makedepends=(
  'git'
  'rust'
  'meson'
  'ninja'
  'cmake'
  'gcc'
  'patchelf'
  'blueprint-compiler'
  'desktop-file-utils'
  'gettext'
)
provides=('karere')
conflicts=('karere')
options=('!lto')

# The CEF binary distribution (built with proprietary codecs) must match the
# `cef` crate pinned in Cargo.lock (currently cef 156.4.0+156.0.26 => CEF
# 156.0.26). If the crate bumps, bump _cef_ver, _cef_tag and the checksums
# together; the author publishes matching CEF builds under the
# `cef-<cef-version>-proprietary-codecs[-N]` tag. prepare() stops the build when
# they no longer match (the app would otherwise crash at start with
# "unsupported CEF API version").
_cef_ver="156.0.26+g8ea0c73+chromium-156.0.8078.88"
_cef_tag="cef-156.0.26-proprietary-codecs-2"
if [[ "$CARCH" == "x86_64" ]]; then
  _cef_platform="linux64"
  _cef_sha256="027473508ec25227972a8bc9182b15723f98cfa2e301746f8e2224c6a58517d5"
elif [[ "$CARCH" == "aarch64" ]]; then
  _cef_platform="linuxarm64"
  _cef_sha256="9dd971ca62b3282b8b6b55da6490b5676550e6b6a25039c599cce9e69f4ccb76"
else
  _cef_platform=""
  _cef_sha256=""
fi
_cef_archive="cef_binary_${_cef_ver}_${_cef_platform}_minimal.zip"

source=(
  "karere::git+https://github.com/tobagin/karere.git"
  "$_cef_archive::https://github.com/tobagin/karere/releases/download/${_cef_tag}/cef_binary_${_cef_ver//+/%2B}_${_cef_platform}_minimal.zip"
)
sha256sums=('SKIP'
            "$_cef_sha256")
noextract=("$_cef_archive")

pkgver() {
  cd "$srcdir/karere"
  git describe --long --tags --match 'v[0-9]*' | sed 's/^v//; s/\([^-]*-g\)/r\1/; s/-/./g'
}

_cef_dir() {
  if compgen -G "$srcdir/cef-src/cef_binary_*" >/dev/null; then
    find "$srcdir/cef-src" -mindepth 1 -maxdepth 1 -type d -name 'cef_binary_*' | head -n1
  else
    printf '%s\n' "$srcdir/cef-src"
  fi
}

prepare() {
  cd "$srcdir/karere"
  # The downloaded CEF must be the version the cef crate was built for.
  local _crate
  _crate="$(awk '/^name = "cef"$/ { getline; print; exit }' Cargo.lock | sed -E 's/.*\+([0-9.]+)".*/\1/')"
  if [[ $_crate != "${_cef_ver%%+*}" ]]; then
    echo "Cargo.lock needs CEF $_crate, but this PKGBUILD downloads ${_cef_ver%%+*}:" \
      "update _cef_ver, _cef_tag and the checksums" >&2
    return 1
  fi
}

build() {
  cd "$srcdir/karere"

  # Keep $srcdir out of the binary (panic paths, file!() strings, debug info).
  export RUSTFLAGS="--remap-path-prefix=$srcdir=/build"

  mkdir -p "$srcdir/cef-src"
  bsdtar -xf "$srcdir/$_cef_archive" -C "$srcdir/cef-src"

  # Stage CEF under a fake root so cef-dll-sys finds it and the build.rs
  # rpath points at the *final* install path (/usr/lib/karere/cef).
  local cef_src cef_stage
  cef_src="$(_cef_dir)"
  cef_stage="$srcdir/cef-staging/usr/lib/karere/cef"
  mkdir -p "$cef_stage"
  cp -a "$cef_src/include" "$cef_src/libcef_dll" "$cef_src/cmake" "$cef_src/CMakeLists.txt" "$cef_stage/"
  cp -a "$cef_src/Release/." "$cef_stage/"
  cp -a "$cef_src/Resources/." "$cef_stage/"
  # cef-dll-sys validates the CEF dir against this file and bails with a
  # version mismatch if the archive name does not parse to a cef version.
  printf '{"type":"minimal","name":"cef_binary_%s_%s_minimal.tar.bz2","sha1":"0000000000000000000000000000000000000000"}\n' \
    "$_cef_ver" "$_cef_platform" > "$cef_stage/archive.json"

  export CEF_PATH="$cef_stage"

  meson setup build \
    --prefix=/usr \
    --buildtype=release \
    -Dprofile=default
  meson compile -C build
}

package() {
  cd "$srcdir/karere"

  # --no-rebuild: meson install would otherwise re-run the cargo custom target
  # inside fakeroot, where makepkg resets RUSTFLAGS, and cargo would rebuild
  # the binary without the $srcdir remap.
  DESTDIR="$pkgdir" meson install -C build --no-rebuild

  local cef_src
  cef_src="$(_cef_dir)"

  install -d "$pkgdir/usr/lib/karere/cef"
  cp -a "$cef_src/Release/." "$pkgdir/usr/lib/karere/cef/"
  cp -a "$cef_src/Resources/." "$pkgdir/usr/lib/karere/cef/"

  # Replace the build-time staging path baked in by build.rs with the real
  # install location, so the dynamic linker finds libcef.so and Chromium's
  # child processes (same binary) resolve their libs too.
  patchelf --set-rpath '$ORIGIN/../lib/karere/cef' "$pkgdir/usr/bin/karere"
}
