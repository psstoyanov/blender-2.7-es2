# Modified version of Blender 2.79b to target ES2 specifically.
# Helpful for SBCs and other limited hardware. 
# Maintainer : bartus <arch-user-repoᘓbartus.33mail.com>
# shellcheck disable=SC2034

#to enforce cuda verison uncomment this line and update value of sm_xx model accordingly
#_cuda_capability+=(sm_30 sm_35 sm_37)
#_cuda_capability+=(sm_50 sm_52 sm_60 sm_61 sm_70 sm_75)
#((TRAVIS)) && _cuda_capability+=(sm_50 sm_52 sm_60 sm_61 sm_70 sm_75) # suppress 3.x to prevent Travis build exceed time limit.

pkgname=blender-2.7-es2
_fragment="#branch=blender2.7"
pkgver=2.79b.r71421.e045fe53f1b
pkgrel=2
pkgdesc="3D modeling, animation, rendering and post-productiom. Blender 2.79b targeting ES2"
arch=('i686' 'x86_64' 'aarch64')
url="https://blender.org/"
depends=('alembic' 'libgl' 'python' 'python-numpy' 'openjpeg' 'desktop-file-utils' 'hicolor-icon-theme'
         'ffmpeg' 'fftw' 'openal' 'freetype2' 'libxi' 'openimageio' 'opencolorio'
         'openvdb' 'opencollada' 'opensubdiv' 'openshadinglanguage' 'libtiff' 'libpng')
makedepends=('git' 'cmake' 'boost' 'mesa' 'llvm')
((DISABLE_NINJA)) || makedepends+=('ninja')

# TODO: Re-test at later date.
# Problematic line. It enforced cuda dependency when it shouldn't have.
((DISABLE_CUDA))  && optdepends=('cuda: CUDA support in Cycles') # || makedepends+=('cuda')

options=(!strip)
provides=('blender-2.7')
conflicts=('blender-2.7')
license=('GPL')
install=blender.install
# NOTE: the source array has to be kept in sync with .gitmodules
# the submodules has to be stored in path ending with git to match
# the path in .gitmodules.
# More info:
#   http://wiki.blender.org/index.php/Dev:Doc/Tools/Git
source=("git://git.blender.org/blender.git${_fragment}"
        'blender-addons.git::git://git.blender.org/blender-addons.git'
        'blender-addons-contrib.git::git://git.blender.org/blender-addons-contrib.git'
        'blender-translations.git::git://git.blender.org/blender-translations.git'
        'blender-dev-tools.git::git://git.blender.org/blender-dev-tools.git'
        blender-2.7.desktop
        SelectCudaComputeArch.patch
        stl_export_iter.patch
        python3.7.patch
        python3.8.patch
        blender.install
        )
sha256sums=('SKIP'
            'SKIP'
            'SKIP'
            'SKIP'
            'SKIP'
            '5d489f24c7401574e79b755aead07c779449ae50a1c3299ad182588127697de1'
            '28e407e3aefdd9bd76805b6033ada0b5b41dd6183bcf4f58a642c109f10c1876'
            '649c21a12a1bfc0207078e1e58b4813a3e898c6dbbbb35d21e1de7c9e8f1985a'
            '47811284f080e38bcfbfb1f7346279245815a064df092989336b0bf3fe4530e9'
            '229853b98bb62e1dec835aea6b2eab4c3dabbc8be591206573a3c1b85f10be59'
            '91543876474c23ac86e5e72962499ffc23a34aa7d803323aa1eda04151feaaf4')

pkgver() {
  cd "$srcdir/blender"
  printf "2.79b.r%s.%s" "$(git rev-list --count HEAD)" "$(git rev-parse --short HEAD)"
}

prepare() {
  cd "$srcdir/blender"
  # update the submodules
  git submodule update --init --recursive --remote
  if [ -z "$_cuda_capability" ] && grep -q nvidia <(lsmod); then 
    git apply -v ${srcdir}/SelectCudaComputeArch.patch
  fi
  git apply -v ${srcdir}/{python3.7,stl_export_iter,python3.8}.patch
}

build() {

  mkdir -p "$srcdir/blender-build"
  cd "$srcdir/blender-build"
  
  _pyver=$(python -c "from sys import version_info; print(\"%d.%d\" % (version_info[0],version_info[1]))")
  msg "python version detected: ${_pyver}"

  # determine whether we can precompile CUDA kernels
  _CUDA_PKG=`pacman -Qq cuda 2>/dev/null` || true
  if [ "$_CUDA_PKG" != "" ] && ! ((DISABLE_CUDA)) ; then
      _EXTRAOPTS=(-DWITH_CYCLES_CUDA_BINARIES=ON \
                  -DCUDA_TOOLKIT_ROOT_DIR=/opt/cuda)
      ((TRAVIS)) && _EXTRAOPTS+=(-DCUDA_NVCC_FLAGS="--ptxas-options=-w")  # suppress all ptxas warnings in Travis build
      if [ "$_cuda_capability" != "" ]; then
        _EXTRAOPTS+=(-DCYCLES_CUDA_BINARIES_ARCH=$(IFS=';'; echo "${_cuda_capability[*]}";))
      fi
  fi

  ((DISABLE_NINJA)) && generator="Unix Makefiles" || generator="Ninja"
  cmake -G "$generator" "$srcdir/blender" \
        -C${srcdir}/blender/build_files/cmake/config/blender_release.cmake \
        -DCMAKE_INSTALL_PREFIX=/usr \
        -DCMAKE_BUILD_TYPE=Release \
        -DWITH_INSTALL_PORTABLE=OFF \
        -DWITH_GL_PROFILE_CORE=OFF \
        -DWITH_GL_PROFILE_ES20=ON \
        -DWITH_SYSTEM_GLEW=ON \
        -DWITH_PYTHON_INSTALL=OFF \
        -DPYTHON_VERSION=${_pyver} \
        -DWITH_LLVM=ON \
        -DBoost_NO_BOOST_CMAKE=ON \
        -DBOOST_ROOT=${LIBDIR}/boost \
        -DBOOST_ROOT=${LIBDIR}/booost/lib/ \
        -DBooost_NO_SYSTEM_PATHS=ON \
        ${_EXTRAOPTS[@]}
  export NINJA_STATUS="[%p | %f<%r<%u | %cbps ] "
  ((DISABLE_NINJA)) && make -j$(nproc) || ninja -d stats
}

package() {
  cd "$srcdir/blender-build"
  ((DISABLE_NINJA)) && make install DESTDIR="$pkgdir" || DESTDIR="$pkgdir" ninja install
  
  msg "add -2.7 sufix to desktop shortcut"
  sed -i 's/=blender/=blender-2.7/g' ${pkgdir}/usr/share/applications/blender.desktop
  sed -i 's/=Blender/=Blender-2.7/g' ${pkgdir}/usr/share/applications/blender.desktop
  mv ${pkgdir}/usr/share/applications/blender.desktop ${pkgdir}/usr/share/applications/blender-2.7.desktop

  msg "add -2.7 sufix to binaries"
  mv ${pkgdir}/usr/bin/blender ${pkgdir}/usr/bin/blender-2.7
  mv ${pkgdir}/usr/bin/blender-thumbnailer.py ${pkgdir}/usr/bin/blender-2.7-thumbnailer.py
#  mv ${pkgdir}/usr/bin/blenderplayer ${pkgdir}/usr/bin/blenderplayer-2.7

  msg "mv doc/blender to doc/blender-2.7"
  mv ${pkgdir}/usr/share/doc/blender ${pkgdir}/usr/share/doc/blender-2.7

  msg "add -2.7 sufix to man page"
  mv ${pkgdir}/usr/share/man/man1/blender.1 ${pkgdir}/usr/share/man/man1/blender-2.7.1

  msg "add -2.7 sufix to all icons"  
  for icon in `find ${pkgdir}/usr/share/icons -type f`
  do
    # ${filename##/*.} extra extenssion from path
    # ${filename%.*} extract filename form path
    # look at bash "manipulatin string" 
    mv $icon ${icon%.*}-2.7.${icon##/*.}
  done

  ## not needed when using options=(!strip)?
  #if [ -e "$pkgdir"/usr/share/blender/*/scripts/addons/cycles/lib/ ] ; then
  #  # make sure the cuda kernels are not stripped
  #  chmod 444 "$pkgdir"/usr/share/blender/*/scripts/addons/cycles/lib/*
  #fi
}
# vim:set sw=2 ts=2 et:
