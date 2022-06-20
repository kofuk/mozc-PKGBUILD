# Maintainer: Koki Fukuda <ko.fu.dev {a} gmail.com>
_build_ibus_mozc=yes
_build_emacs_mozc=yes
_build_mozc_tool=yes

_build_qt_renderer=auto

pkgname=('mozc')
pkgver=2.28.4750.100
pkgrel=1
# Git commit ID
_vc_rev='6d223e2c7f55f22cb71b55a93129a4c721c2965b'
arch=('x86_64')
url='https://github.com/google/mozc'
license=('BSD' 'custom')
groups=('mozc-im')
makedepends=('bazel' 'git' 'java-environment=11')
source=(
    "mozc::git+https://github.com/google/mozc.git#commit=${_vc_rev}"
    'emoji-13-0.tsv'
    'emoji-13-1.tsv'
    'emoji-14-0.tsv'
    'emoji-misc.tsv'
)
sha256sums=(
    'SKIP'
    'SKIP'
    'SKIP'
    'SKIP'
    'SKIP'
)

if [ "${_build_qt_renderer}" = 'auto' ]; then
    if [ "${_build_ibus_mozc}" = 'yes' ]; then
        _build_qt_renderer=no
    else
        _build_qt_renderer=yes
        makedepends+=('qt5-base')
    fi
fi

if [ "${_build_ibus_mozc}" = 'yes' ]; then
    pkgname+=('ibus-mozc')
    makedepends+=('ibus>=1.4.1')
fi

if [ "${_build_emacs_mozc}" = 'yes' ]; then
    pkgname+=('emacs-mozc')
fi

if [ "${_build_mozc_tool}" = 'yes' ]; then
    pkgname+=('mozc-tool')
    makedepends+=('qt5-base')
fi

pkgver() {
    grep -E '^(MAJOR|MINOR|BUILD_OSS|REVISION) ' "${srcdir}/mozc/src/data/version/mozc_version_template.bzl" |
        sed -E 's/[A-Z_]+ *= *([0-9]+)/\1./g' | tr -d '\n' | sed 's/.$//'
}

prepare() {
    cd "${srcdir}/mozc"

    git submodule update --init --recursive

    # Remove Android deps
    sed -Ei 's/^android_/#&/g' "${srcdir}/mozc/src/WORKSPACE.bazel"

    # Fix GLib path
    sed -Ei 's@lib/x86_64-linux-gnu/glib-2.0@lib/glib-2.0@g' "${srcdir}/mozc/src/BUILD.ibus.bazel"
    # Fix Qt path
    sed -Ei 's@/usr/include/x86_64-linux-gnu/qt5@/usr/include/qt@g' "${srcdir}/mozc/src/config.bzl"

    # Add emoji entries (because upstream doesn't support newer emoji)
    cat "${srcdir}/emoji-13-0.tsv" "${srcdir}/emoji-13-1.tsv" \
        "${srcdir}/emoji-14-0.tsv" "${srcdir}/emoji-misc.tsv" \
        >>"${srcdir}/mozc/src/data/emoji/emoji_data.tsv"
}

build() {
    cd "${srcdir}/mozc/src"

    _targets=()

    for _pkg in "${pkgname[@]}"; do
        case "${_pkg}" in
            mozc)
                _targets+=('//server:mozc_server')
                if [ "${_build_qt_renderer}" = 'yes' ]; then
                    _targets+=('//renderer:mozc_renderer')
                fi
                ;;
            ibus-mozc)
                _targets+=('//unix/ibus:ibus_mozc' '//unix:icons')
                ;;
            mozc-tool)
                _targets+=('//gui/tool:mozc_tool')
                ;;
            emacs-mozc)
                _targets+=('//unix/emacs:mozc_emacs_helper')
                ;;
        esac
    done

    _orig_java="$(archlinux-java get)"
    if [ "${_orig_java}" != 'java-11-openjdk' ]; then
        # This is not polite, but it's needed to execute bazel.
        echo "Switching Java environment to java-11-openjdk (from ${_orig_java})..."
        sudo archlinux-java set java-11-openjdk
    fi

    echo "Building following targets: ${_targets[*]}"
    bazel build "${_targets[@]}" --config oss_linux -c opt
    bazel shutdown

    if [ -e "${srcdir}/mozc/src/bazel-bin/unix/ibus/mozc.xml" ]; then
        # Fill version field for IBus component
        sed -i "s/0.0.0.0/${pkgver}/" "${srcdir}/mozc/src/bazel-bin/unix/ibus/mozc.xml"
    fi

    if [ "${_orig_java}" != "$(archlinux-java get)" ]; then
        echo 'Reverting Java environment...'
        sudo archlinux-java set "${_orig_java}"
    fi
}

# Mozc base package
package_mozc() {
    pkgdesc='A Japanese Input Method Editor designed for multi-platform'
    [ "${_build_qt_renderer}" = 'yes' ] && \
        depends=('qt5-base')

    cd "${srcdir}/mozc/src/bazel-bin"
    install -D -m 755 'server/mozc_server' "${pkgdir}/usr/lib/mozc/mozc_server"
    [ "${_build_qt_renderer}" = 'yes' ] && \
        install -D -m 755 'renderer/mozc_renderer' "${pkgdir}/usr/lib/mozc/mozc_renderer"

    install -d "${pkgdir}/usr/share/licenses/mozc/"
    cd "${srcdir}/mozc"
    install -m 644 LICENSE src/data/installer/*.html "${pkgdir}/usr/share/licenses/mozc/"
}

# IBus component package
package_ibus-mozc() {
    pkgdesc='IBus engine for Mozc'

    depends=("mozc=${pkgver}" 'ibus>=1.4.1')

    cd "${srcdir}/mozc/src/bazel-bin"
    install -D -m 755 'unix/ibus/ibus_mozc' "${pkgdir}/usr/lib/ibus-mozc/ibus-engine-mozc"
    install -D -m 644 'unix/ibus/mozc.xml' "${pkgdir}/usr/share/ibus/component/mozc.xml"

    cd "${srcdir}/mozc/src"
    install -D -m 644 'data/images/unix/ime_product_icon_opensource-32.png' "${pkgdir}/usr/share/ibus-mozc/product_icon.png"
    install -D -m 644 'data/images/unix/ui-tool.png'             "${pkgdir}/usr/share/ibus-mozc/tool.png"
    install -D -m 644 'data/images/unix/ui-properties.png'       "${pkgdir}/usr/share/ibus-mozc/properties.png"
    install -D -m 644 'data/images/unix/ui-dictionary.png'       "${pkgdir}/usr/share/ibus-mozc/dictionary.png"
    install -D -m 644 'data/images/unix/48x48/direct.png'        "${pkgdir}/usr/share/ibus-mozc/direct.png"
    install -D -m 644 'data/images/unix/48x48/hiragana.png'      "${pkgdir}/usr/share/ibus-mozc/hiragana.png"
    install -D -m 644 'data/images/unix/48x48/katakana_half.png' "${pkgdir}/usr/share/ibus-mozc/katakana_half.png"
    install -D -m 644 'data/images/unix/48x48/katakana_full.png' "${pkgdir}/usr/share/ibus-mozc/katakana_full.png"
    install -D -m 644 'data/images/unix/48x48/alpha_half.png'    "${pkgdir}/usr/share/ibus-mozc/alpha_half.png"
    install -D -m 644 'data/images/unix/48x48/alpha_full.png'    "${pkgdir}/usr/share/ibus-mozc/alpha_full.png"
}

package_mozc-tool() {
    pkgdesc='A Japanese Input Method Editor designed for multi-platform (Settings GUI)'
    depends=("mozc=${pkgver}" 'qt5-base')

    cd "${srcdir}/mozc/src/bazel-bin"
    install -D -m 755 'gui/tool/mozc_tool' "${pkgdir}/usr/lib/mozc/mozc_tool"
}

# Emacs helper module package
package_emacs-mozc() {
    pkgdesc='Mozc for Emacs'

    depends=("mozc=${pkgver}" 'emacs')
    conflicts=('emacs-mozc-bin')

    cd "${srcdir}/mozc/src/bazel-bin"
    install -D -m 755 'unix/emacs/mozc_emacs_helper' "${pkgdir}/usr/bin/mozc_emacs_helper"
    install -D -m 644 "${srcdir}/mozc/src/unix/emacs/mozc.el" "${pkgdir}/usr/share/emacs/site-lisp/emacs-mozc"
}
