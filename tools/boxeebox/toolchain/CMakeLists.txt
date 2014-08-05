cmake_minimum_required(VERSION 2.8)

set(TARGET "i686-unknown-linux-gnu")
set(TARGET_DIR "${SYSROOT}/usr")

set(BINUTILS_VERSION "2.24" CACHE STRING "Version of binutils to build")
set(GCC_VERSION "4.9.1" CACHE STRING "Version of GCC to build")
set(ENABLE_GDB OFF CACHE BOOL "Wether to build gdb or not")

include(ExternalProject)

execute_process(COMMAND ${CMAKE_C_COMPILER} -dumpmachine OUTPUT_VARIABLE HOSTMACHINE)
string(REPLACE "\n" "" HOSTMACHINE ${HOSTMACHINE})

ExternalProject_Add(
	kernel
	URL					ftp://ftp.kernel.org/pub/linux/kernel/v3.x/linux-3.15.4.tar.gz
	URL_MD5				d530ecf5197f9fc00ebcf3a8203e2ade
	PATCH_COMMAND		cp ${CMAKE_SOURCE_DIR}/generic-linux.i386.conf .config
	CONFIGURE_COMMAND   ""
        COMMAND         make mrproper	
	BUILD_COMMAND		make ARCH=i386 INSTALL_HDR_PATH=dest headers_install
	INSTALL_COMMAND     ${CMAKE_COMMAND} -E copy_directory dest/include ${TARGET_DIR}/include
	BUILD_IN_SOURCE		1
)

ExternalProject_Add(
	gmp
	URL 				https://gmplib.org/download/gmp/gmp-6.0.0a.tar.bz2
	URL_MD5				b7ff2d88cae7f8085bd5006096eed470
	CONFIGURE_COMMAND 	LDFLAGS=-Wl,-rpath,${TARGET_DIR}/lib <SOURCE_DIR>/configure ABI=32 --enable-cxx --prefix=${TARGET_DIR} --build=${HOSTMACHINE}
	BUILD_COMMAND 		make ${PARALLEL}
	INSTALL_COMMAND     make install
	DEPENDS 			kernel
)

ExternalProject_Add(
	mpfr
	URL 				http://ftp.gnu.org/gnu/mpfr/mpfr-3.1.2.tar.bz2
	URL_MD5				ee2c3ac63bf0c2359bf08fc3ee094c19
	CONFIGURE_COMMAND 	LDFLAGS=-Wl,-rpath,${TARGET_DIR}/lib <SOURCE_DIR>/configure --prefix=${TARGET_DIR} --target=${TARGET} --enable-shared --with-gmp-lib=${TARGET_DIR}/lib --with-gmp-include=${TARGET_DIR}/include
	BUILD_COMMAND 		make ${PARALLEL}
	INSTALL_COMMAND     make install
	DEPENDS 			gmp
)

ExternalProject_Add(
	mpc
	URL 				http://ftp.gnu.org/gnu/mpc/mpc-1.0.2.tar.gz
	URL_MD5				68fadff3358fb3e7976c7a398a0af4c3
	CONFIGURE_COMMAND 	LDFLAGS=-Wl,-rpath,${TARGET_DIR}/lib <SOURCE_DIR>/configure --prefix=${TARGET_DIR} --target=${TARGET} --enable-shared --disable-static --with-gmp=${TARGET_DIR} --with-mpfr=${TARGET_DIR}
	BUILD_COMMAND 		make ${PARALLEL}
	INSTALL_COMMAND     make install
	DEPENDS 			gmp mpfr
)

ExternalProject_Add(
	boot-binutils
	PREFIX 				${CMAKE_BINARY_DIR}/binutils
	URL 				http://ftp.gnu.org/gnu/binutils/binutils-${BINUTILS_VERSION}.tar.bz2
	URL_MD5 			e0f71a7b2ddab0f8612336ac81d9636b
	CONFIGURE_COMMAND	<SOURCE_DIR>/configure --target=${TARGET} --with-sysroot=${SYSROOT} --with-lib-path=${TARGET_DIR}/lib --prefix=${TARGET_DIR} --disable-nls --disable-werror --with-gmp=${TARGET_DIR} --with-mpfr=${TARGET_DIR} --with-mpc=${TARGET_DIR} --disable-multilib  --disable-libada --disable-libssp --enable-version-specific-runtime-libs --enable-gold --enable-lto --program-prefix=${TARGET}-
	BUILD_COMMAND		make ${PARALLEL}
	DEPENDS 			mpc
)

ExternalProject_Add(
	boot-gcc
	PREFIX 				${CMAKE_BINARY_DIR}/gcc
	URL 				ftp://ftp.mirrorservice.org/sites/sourceware.org/pub/gcc/releases/gcc-${GCC_VERSION}/gcc-${GCC_VERSION}.tar.bz2
	URL_MD5 			fddf71348546af523353bd43d34919c1
	PATCH_COMMAND       sed -i "/k prot/agcc_cv_libc_provides_ssp=yes" ${CMAKE_BINARY_DIR}/gcc/src/boot-gcc/configure
	CONFIGURE_COMMAND	<SOURCE_DIR>/configure --target=${TARGET} --prefix=${TARGET_DIR} --with-sysroot=${SYSROOT} --disable-nls --without-headers --with-newlib --enable-languages=c --with-gmp=${TARGET_DIR}/include --with-mpfr=${TARGET_DIR}/include --with-mpc=${TARGET_DIR}/include --enable-bootstrap --with-native-system-header-dir=${TARGET_DIR}/include --enable-checking=release  --disable-libquadmath  --disable-libatomic --disable-decimal-float --disable-shared --disable-threads --disable-libmudflap --disable-libssp --disable-libgomp --disable-multilib --disable-__cxa_atexit --disable-libsanitizer --disable-libitm
	BUILD_COMMAND		make all-gcc all-target-libgcc || return 1 ${PARALLEL}
	INSTALL_COMMAND make install-gcc install-target-libgcc || return 1
		COMMAND ${CMAKE_COMMAND} -E create_symlink ${TARGET_DIR}/lib/gcc/${TARGET}/${GCC_VERSION}/libgcc.a  ${TARGET_DIR}/lib/gcc/${TARGET}/${GCC_VERSION}/libgcc_eh.a
	DEPENDS 			boot-binutils
)

ExternalProject_Add(
	glibc
	URL 				http://ftp.gnu.org/gnu/glibc/glibc-2.19.tar.gz
	URL_MD5 			5374d29864b583622b62bfc6b8429418
	PATCH_COMMAND		cat elf/Makefile | sed s,\ readelf\ ,\ ${PREFIX}/bin/${TARGET}-readelf\ ,g > elf/Makefile.new
		COMMAND 		${CMAKE_COMMAND} -E copy elf/Makefile.new elf/Makefile
		COMMAND 		cat configure | sed s,\ readelf\ ,\ ${TARGET_DIR}/bin/${TARGET}-readelf\ ,g > configure.new
		COMMAND 		${CMAKE_COMMAND} -E copy configure.new configure
		COMMAND 		chmod +x configure
		COMMAND         sed -i s|<rpc/types.h>|"rpc/types.h"| sunrpc/rpc_clntout.c
	CONFIGURE_COMMAND	libc_cv_forced_unwind=yes libc_cv_c_cleanup=yes libc_cv_ctors_header=yes libc_cv_ssp=no <SOURCE_DIR>/configure --prefix=/usr --target=${TARGET} --host=${TARGET} --with-sysroot=${SYSROOT} --with-lib-path=${TARGET_DIR}/lib --with-binutils=${TARGET_DIR}/bin --disable-profile --enable-bind-now --with-elf --with-tls --cache-file=config.cache --enable-obsolete-rpc --without-gd --without-selinux --disable-nls --enable-kernel=2.6 --with-kernel-headers=${TARGET_DIR}/include --with-headers=${TARGET_DIR}/include CFLAGS=-O2\ -fno-stack-protector\ -U_FORTIFY_SOURCE\ -march=i686 --with-arch-32=i686 --disable-multi-arch --program-prefix=${TARGET}-
	BUILD_COMMAND		make ${PARALLEL}
	INSTALL_COMMAND		make ${PARALLEL} install_root=${SYSROOT} install
	DEPENDS boot-gcc
)

ExternalProject_Add(
	binutils
	PREFIX 				${CMAKE_BINARY_DIR}/binutils
	URL 				http://ftp.gnu.org/gnu/binutils/binutils-${BINUTILS_VERSION}.tar.bz2
	URL_MD5 			e0f71a7b2ddab0f8612336ac81d9636b
	CONFIGURE_COMMAND	<SOURCE_DIR>/configure --target=${TARGET} --with-sysroot --with-lib-path=${TARGET_DIR}/lib --prefix=${TARGET_DIR} --disable-nls --disable-werror --with-gmp=${TARGET_DIR} --with-mpfr=${TARGET_DIR} --with-mpc=${TARGET_DIR} --with-arch-32=i686 --disable-multilib  --disable-libada --disable-libssp --enable-version-specific-runtime-libs --enable-gold --enable-lto --program-prefix=${TARGET}-
	BUILD_COMMAND		make ${PARALLEL}
	INSTALL_COMMAND		make ${PARALLEL} install_root=${SYSROOT} install
	DEPENDS 			glibc
)

ExternalProject_Add(
	gcc2
	PREFIX 				${CMAKE_BINARY_DIR}/gcc
	URL 				ftp://ftp.mirrorservice.org/sites/sourceware.org/pub/gcc/releases/gcc-${GCC_VERSION}/gcc-${GCC_VERSION}.tar.bz2
	URL_MD5 			fddf71348546af523353bd43d34919c1
	PATCH_COMMAND       sed -i "/k prot/agcc_cv_libc_provides_ssp=yes" ${CMAKE_BINARY_DIR}/gcc/src/gcc2/configure
	CONFIGURE_COMMAND	<SOURCE_DIR>/configure --enable-languages=c,c++ --target=${TARGET} --with-sysroot=${SYSROOT} --prefix=${TARGET_DIR} --with-local-prefix=${TARGET_DIR} --build=${TARGET} --with-lib-path=${TARGET_DIR}/lib --with-system-zlib --with-arch-32=i686 --with-gmp=${TARGET_DIR}/include --with-mpfr=${TARGET_DIR}/include --with-mpc=${TARGET_DIR}/include --enable-lto --cache-file=config.cache --disable-werror --disable-libstdcxx-pch --with-gnu-as --with-gnu-ld --enable-clocale=gnu --enable-shared --enable-threads=posix --enable-plugin --disable-nls --disable-multilib --disable-libgomp --disable-libmudflap --disable-libssp --disable-libquadmath --disable-bootstrap --enable-checking=release --with-ppl=no --with-cloog=no --program-prefix=${TARGET}-
	BUILD_COMMAND		make all-gcc all-target-libgcc all-target-libstdc++-v3 || return 1 ${PARALLEL}
	INSTALL_COMMAND     make install-gcc install-target-libgcc install-target-libstdc++-v3 install_root=${SYSROOT} || return 1
	DEPENDS 			binutils
)

ExternalProject_Add(
	zlib
	URL                 http://zlib.net/zlib-1.2.8.tar.gz
	URL_MD5             44d667c142d7cda120332623eab69f40
	CONFIGURE_COMMAND	CROSS_PREFIX=${TARGET} AR=${TARGET}-ar CC=${TARGET}-gcc RANLIB=${TARGET}-ranlib <SOURCE_DIR>/configure --uname=Linux --prefix=${SYSROOT}/usr
	BUILD_COMMAND 		""
	INSTALL_COMMAND 	make ${PARALLEL} install
	BUILD_IN_SOURCE 	1
	DEPENDS 			gcc2
)

if(ENABLE_GDB)
ExternalProject_Add(
	gdb
	PREFIX 				${CMAKE_BINARY_DIR}/gdb
	URL 				http://ftp.gnu.org/gnu/gdb/gdb-6.8a.tar.bz2
	URL_MD5				da20d043e6729f74b909bd507bcae5c9
	CONFIGURE_COMMAND 	<SOURCE_DIR>/configure --with-sysroot=${SYSROOT} --target=${TARGET} --prefix=${TARGET_DIR} --program-prefix=${TARGET}- --disable-werror
	BUILD_COMMAND 		make ${PARALLEL}
	DEPENDS 			gcc2
)

ExternalProject_Add(
	termcap
	URL 				ftp://ftp.gnu.org/gnu/termcap/termcap-1.3.1.tar.gz
	URL_MD5 			ffe6f86e63a3a29fa53ac645faaabdfa
	CONFIGURE_COMMAND 	CC=${TARGET}-gcc RANLIB=${TARGET}-ranlib <SOURCE_DIR>/configure --target=${TARGET} --prefix=${SYSROOT}/usr/local/
	BUILD_COMMAND 		make ${PARALLEL}
	DEPENDS 			gcc2
)

ExternalProject_Add(
	targetgdb
	PREFIX 				${CMAKE_BINARY_DIR}/gdb
	URL 				http://ftp.gnu.org/gnu/gdb/gdb-6.8a.tar.bz2
	URL_MD5				da20d043e6729f74b909bd507bcae5c9
	CONFIGURE_COMMAND 	LDFLAGS=-L${SYSROOT}/usr/local/lib <SOURCE_DIR>/configure --host=${TARGET} --prefix=${SYSROOT}/usr/local/ --disable-werror
	BUILD_COMMAND 		make ${PARALLEL}
	DEPENDS 			gcc2 termcap
)
endif()

file(WRITE "${SYSROOT}/usr/bin/${TARGET}-pkg-config" "
#!/bin/bash

set -e
export PKG_CONFIG_DIR=
export PKG_CONFIG_LIBDIR=${SYSROOT}/usr/lib/pkgconfig:${SYSROOT}/usr/share/pkgconfig:${SYSROOT}/usr/local/lib/pkgconfig:${SYSROOT}/usr/local/share/pkgconfig
export PKG_CONFIG_SYSROOT_DIR=${SYSROOT}

exec pkg-config \"\$@\" | python -c \"import sys; print sys.stdin.read().replace('${SYSROOT}${SYSROOT}', '${SYSROOT}'),\" && test \${PIPESTATUS[0]} -eq 0
")
execute_process(COMMAND chmod +x ${SYSROOT}/usr/bin/${TARGET}-pkg-config)
