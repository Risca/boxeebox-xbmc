cmake_minimum_required(VERSION 2.8)

set(TARGET "i686-pc-linux-gnu")
set(TARGET_DIR "${SYSROOT}/usr")

set(BINUTILS_VERSION "2.20.1a" CACHE STRING "Version of binutils to build")
set(GCC_VERSION "4.4.3" CACHE STRING "Version of GCC to build")
set(ENABLE_GDB OFF CACHE BOOL "Wether to build gdb or not")

include(ExternalProject)

execute_process(COMMAND ${CMAKE_C_COMPILER} -dumpmachine OUTPUT_VARIABLE HOSTMACHINE)
string(REPLACE "\n" "" HOSTMACHINE ${HOSTMACHINE})

ExternalProject_Add(
	kernel
	URL					ftp://ftp.kernel.org/pub/linux/kernel/v2.6/linux-2.6.32.8.tar.bz2
	URL_MD5				82023ede52f067fcc55c5e70b02e48ae
	PATCH_COMMAND		cp ${CMAKE_SOURCE_DIR}/kernel.config .config
	CONFIGURE_COMMAND	""
	BUILD_COMMAND		make ARCH=i386 include/linux/version.h
	INSTALL_COMMAND		${CMAKE_COMMAND} -E copy_directory include/linux ${TARGET_DIR}/include/linux
		COMMAND			${CMAKE_COMMAND} -E copy_directory arch/x86/include/asm ${TARGET_DIR}/include/asm
		COMMAND			${CMAKE_COMMAND} -E copy_directory include/asm-generic ${TARGET_DIR}/include/asm-generic
	BUILD_IN_SOURCE		1
)

ExternalProject_Add(
	boot-binutils
	PREFIX 				${CMAKE_BINARY_DIR}/binutils
	URL 				http://ftp.gnu.org/gnu/binutils/binutils-${BINUTILS_VERSION}.tar.bz2
	URL_MD5 			2b9dc8f2b7dbd5ec5992c6e29de0b764
	PATCH_COMMAND       patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/100-ppc64-pie.patch
	    COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/110-RPATH_ENVVAR-smack.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/120-sh-targets.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/130-ld-sysroot.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/140-check_ldrunpath_length.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/150-pt-pax-flags-20090909.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/160-amd64-32bit-path.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/170-warn-textrel.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/190-use-new-ld-dtags.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/200-document-new-dtags-behaviour.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/220-use-relro.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/230-libiberty-pic.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/240-ld_makefile.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/250-arm-eabi-conf.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/270-better_file_error.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/280-unexport_LD_LIBRARY_PATH.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/290-ld-stub-crash.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/BINUTILS-2.20.1a-patches/300-mips64-defaultemul.patch
	CONFIGURE_COMMAND	 <SOURCE_DIR>/configure --with-sysroot=${SYSROOT} --with-lib-path=${TARGET_DIR}/lib --prefix=${TARGET_DIR} --target ${TARGET} --disable-nls --disable-werror --program-prefix=${TARGET}-
	BUILD_COMMAND		make ${PARALLEL}
	DEPENDS 			kernel
)

ExternalProject_Add(
	boot-gcc
	PREFIX 				${CMAKE_BINARY_DIR}/gcc
	URL 				ftp://ftp.mirrorservice.org/sites/sourceware.org/pub/gcc/releases/gcc-${GCC_VERSION}/gcc-${GCC_VERSION}.tar.bz2
	URL_MD5 			fe1ca818fc6d2caeffc9051fe67ff103
	CONFIGURE_COMMAND	<SOURCE_DIR>/configure --with-sysroot=${SYSROOT} --target=${TARGET} --prefix=${TARGET_DIR} --disable-nls --without-headers --with-arch-32=i686 --with-newlib --enable-languages=c --disable-decimal-float --disable-shared --disable-threads --disable-libmudflap --disable-libssp --disable-libgomp --with-local-prefix=${TARGET_DIR} --with-native-system-header-dir=${TARGET_DIR}/include --enable-bootstrap --disable-libsanitizer --enable-checking=release  #--disable-libquadmath  --disable-libatomic
	PATCH_COMMAND       patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/100-alpha-mieee-default.patch
	    COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/110-trampolinewarn.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/120-java-nomulti.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/130-cross-compile.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/140-default-format-security.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/150-default-fortify-source.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/160-netbsd-symbolic.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/170-sparc64-bsd.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/180-libgomp-no-werror.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/190-flatten-switch-stmt-00.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/200-libiberty.h-asprintf.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/210-arm-unbreak-armv4t.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/220-libiberty-pic.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/230-superh-default-multilib.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/240-libstdc++-pic.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/250-ia64-noteGNUstack.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/260-sh-libgcc-stacks.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/270-sh-pr24836.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/280-freebsd.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/290-freebsd.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/300-uclibc-conf.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/310-missing-execinfo_h.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/320-c99-snprintf.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/330-libmudflap-susv3-legacy.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/340-ecjx-host-cc.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/350-index_macro.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/360-arm-bigendian.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/370-bootstrap-target_lib_path.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/380-powerpc-libgcc_s-link-libm.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/390-arm-softfloat-libgcc.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/400-pr42289-fix-libffi-build-on-arm-oabi.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/410-libgcc_eh.a.patch
	BUILD_COMMAND		make all-gcc all-target-libgcc || return 1 ${PARALLEL}
	INSTALL_COMMAND make install-gcc install-target-libgcc || return 1
		COMMAND         ln -sf ${TARGET_DIR}/lib/gcc/${TARGET}/${GCC_VERSION}/libgcc.a ${TARGET_DIR}/lib/gcc/${TARGET}/${GCC_VERSION}/libgcc_eh.a
		COMMAND         rm -rf ${TARGET_DIR}/lib/libgcc_s.so
        COMMAND         ln -sf libgcc_s.so.1 ${TARGET_DIR}/lib/libgcc_s.so
	DEPENDS 			boot-binutils
)

ExternalProject_Add(
	glibc
	URL 				http://ftp.gnu.org/gnu/glibc/glibc-2.11.1.tar.gz
	URL_MD5 			cf60820c052b14a9c5a7816fc7dc0161
	PATCH_COMMAND       patch -p1 -N < ${CMAKE_SOURCE_DIR}/GLIBC_2.11.1-patches/900-march-i686.patch
	    COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GLIBC_2.11.1-patches/910-typedef-caddr.patch
	CONFIGURE_COMMAND	CC=${TARGET}-gcc <SOURCE_DIR>/configure --target=${TARGET} --host=${TARGET} --prefix=/usr --with-sysroot=${SYSROOT} --with-elf --enable-obsolete-rpc --disable-nls --enable-kernel=2.6 -with-kernel-headers=/usr/include --with-headers=${TARGET_DIR}/include --with-binutils=${TARGET_DIR}/bin libc_cv_forced_unwind=yes libc_cv_c_cleanup=yes libc_cv_ctors_header=yes CFLAGS=-O2\ -U_FORTIFY_SOURCE\ -march=i686 --disable-sanity-checks
	BUILD_COMMAND		make
	INSTALL_COMMAND		make install_root=${SYSROOT} install
	DEPENDS boot-gcc
)

ExternalProject_Add(
	gcc2
	PREFIX 				${CMAKE_BINARY_DIR}/gcc
	URL 				ftp://ftp.mirrorservice.org/sites/sourceware.org/pub/gcc/releases/gcc-${GCC_VERSION}/gcc-${GCC_VERSION}.tar.bz2
	URL_MD5 			fe1ca818fc6d2caeffc9051fe67ff103
	PATCH_COMMAND       patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/100-alpha-mieee-default.patch
	    COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/110-trampolinewarn.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/120-java-nomulti.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/130-cross-compile.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/140-default-format-security.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/150-default-fortify-source.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/160-netbsd-symbolic.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/170-sparc64-bsd.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/180-libgomp-no-werror.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/190-flatten-switch-stmt-00.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/200-libiberty.h-asprintf.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/210-arm-unbreak-armv4t.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/220-libiberty-pic.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/230-superh-default-multilib.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/240-libstdc++-pic.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/250-ia64-noteGNUstack.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/260-sh-libgcc-stacks.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/270-sh-pr24836.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/280-freebsd.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/290-freebsd.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/300-uclibc-conf.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/310-missing-execinfo_h.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/320-c99-snprintf.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/330-libmudflap-susv3-legacy.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/340-ecjx-host-cc.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/350-index_macro.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/360-arm-bigendian.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/370-bootstrap-target_lib_path.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/380-powerpc-libgcc_s-link-libm.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/390-arm-softfloat-libgcc.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/400-pr42289-fix-libffi-build-on-arm-oabi.patch
		COMMAND         patch -p1 -N < ${CMAKE_SOURCE_DIR}/GCC-4.4.3-patches/410-libgcc_eh.a.patch
	CONFIGURE_COMMAND	<SOURCE_DIR>/configure --with-sysroot=${SYSROOT} --target=${TARGET} --prefix=${TARGET_DIR} --disable-libada -enable-c99 --disable-libstdcxx-pch --enable-clocale=gnu --enable-shared --enable-threads=posix --enable-__cxa_atexit --enable-languages=c,c++ --disable-nls --disable-multilib --disable-libgomp --disable-libmudflap --disable-libssp --disable-bootstrap --enable-checking=release --with-local-prefix=${TARGET_DIR} --with-native-system-header-dir=${TARGET_DIR}/include --program-prefix=${TARGET}-
	BUILD_COMMAND 		make ${PARALLEL}
	DEPENDS 			glibc
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
