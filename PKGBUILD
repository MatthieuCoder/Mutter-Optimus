pkgbase=mutter-optimus
if [ -n "$_disable_docs" ]; then
    pkgname=mutter-optimus
else
    pkgname=(mutter-optimus mutter-optimus-docs)
fi
epoch=1
pkgver=45.0
pkgrel=1
pkgdesc="A window manager for GNOME | Attempts to improve performances with non-upstreamed merge-requests and frequent stable branch resync"
url="https://gitlab.gnome.org/GNOME/mutter"
arch=(x86_64 aarch64)
license=(GPL)
depends=(
    colord
    dconf
    gnome-desktop-4
    gnome-settings-daemon
    graphene
    gsettings-desktop-schemas
    iio-sensor-proxy
    lcms2
    libcanberra
    libgudev
    libinput
    libsm
    libsysprof-capture
    libxkbcommon-x11
    libxkbfile
    pipewire
    startup-notification
    xorg-xwayland
)
makedepends=(
    egl-wayland
    gi-docgen
    git
    gobject-introspection
    gtk3
    meson
    sysprof
    wayland-protocols
    xorg-server
    xorg-server-xvfb
)

checkdepends=(xorg-server-xvfb pipewire-session-manager python-dbusmock zenity)
_commit=4f6c91847088d7d6476b88575b3a6601b819b443  # tags/45.0^0
source=("$pkgname::git+https://gitlab.gnome.org/GNOME/mutter.git#commit=$_commit")
sha256sums=('SKIP')

pkgver() {
    cd $pkgname
    git describe --tags | sed 's/[a^-]*-g/r&/;s/-/+/g'
}

pick_mr() {
    echo "Downloading then Merging $1..."
    curl -O "https://gitlab.gnome.org/GNOME/mutter/-/merge_requests/$1.diff"
    git apply "$1.diff"
}

prepare() {
    cd $pkgname
    
    git reset --hard
    git cherry-pick --abort || true
    git clean -fd
    
    pick_mr "1441"
    pick_mr "3301"
}

build() {
    local meson_options=(
        -D egl_device=true
        -D wayland_eglstream=true
        -D installed_tests=false
        -D docs=$(if ! [ -n "$_disable_docs" ]; then echo "true"; else echo "false"; fi)
        -D tests=$(if [ -n "$_enable_check" ]; then echo "true"; else echo "false"; fi)
    )
    
    CFLAGS="${CFLAGS/-O2/-O3} -fno-semantic-interposition"
    LDFLAGS+=" -Wl,-Bsymbolic-functions"
    
    arch-meson $pkgname build "${meson_options[@]}"
    meson compile -C build
}

_check() (
    mkdir -p -m 700 "${XDG_RUNTIME_DIR:=$PWD/runtime-dir}"
    glib-compile-schemas "${GSETTINGS_SCHEMA_DIR:=$PWD/build/data}"
    export XDG_RUNTIME_DIR GSETTINGS_SCHEMA_DIR
    local _pipewire_session_manager=$(pacman -Qq pipewire-session-manager)
    
    pipewire &
    _p1=$!
    
    $_pipewire_session_manager &
    _p2=$!
    
    trap "kill $_p1 $_p2; wait" EXIT
    
    meson test -C build --print-errorlogs -t 3
)

if [ -n "$_enable_check" ]; then
    check() {
        echo "Tests may be broken with certain setups. Use with caution!"
        dbus-run-session xvfb-run -s '-nolisten local +iglx -noreset' \
        bash -c "$(declare -f _check); _check"
    }
fi

_pick() {
    local p="$1" f d; shift
    for f; do
        d="$srcdir/$p/${f#$pkgdir/}"
        mkdir -p "$(dirname "$d")"
        mv "$f" "$d"
        rmdir -p --ignore-fail-on-non-empty "$(dirname "$f")"
    done
}

package_mutter-optimus() {
    provides=(mutter libmutter-13.so)
    conflicts=(mutter)
    groups=(gnome)
    
    meson install -C build --destdir "$pkgdir"
    
    if ! [ -n "$_disable_docs" ]; then
        _pick docs "$pkgdir"/usr/share/mutter-*/doc
    fi
}

if ! [ -n "$_disable_docs" ]; then
    package_mutter-optimus-docs() {
        provides=(mutter-docs)
        conflicts=(mutter-docs)
        pkgdesc+=" (documentation)"
        depends=()
        
        mv docs/* "$pkgdir"
    }
fi
