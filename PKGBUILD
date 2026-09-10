# Maintainer: MicLeh <micleh at proton dot me>

pkgname=karere-git
pkgver=4.2.5.r0.gc2cff09
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
# `cef` crate pinned in Cargo.lock (currently cef 150.0.0+150.0.10 => CEF
# 150.0.10). If the crate bumps, bump _cef_ver together; the author publishes
# matching CEF builds under the `cef-<cef-version>-proprietary-codecs` tag.
_cef_ver="150.0.10+g8042e43+chromium-150.0.7871.101"
if [[ "$CARCH" == "x86_64" ]]; then
  _cef_platform="linux64"
  _cef_sha256="3bbe298368c4d87c19ad9b7ed4e8449ea91b32ffa3cefc8672791a1b96c9c3b9"
elif [[ "$CARCH" == "aarch64" ]]; then
  _cef_platform="linuxarm64"
  _cef_sha256="543bc10ce854fc39b0493ff8b369c47d65fa43f2fb974b734252eea805bffc53"
else
  _cef_platform=""
  _cef_sha256=""
fi
_cef_archive="cef_binary_${_cef_ver}_${_cef_platform}_minimal.zip"

source=(
  "karere::git+https://github.com/tobagin/karere.git"
  "$_cef_archive::https://github.com/tobagin/karere/releases/download/cef-150.0.10-proprietary-codecs/cef_binary_${_cef_ver//+/%2B}_${_cef_platform}_minimal.zip"
)
sha256sums=('SKIP'
            '3bbe298368c4d87c19ad9b7ed4e8449ea91b32ffa3cefc8672791a1b96c9c3b9')
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
