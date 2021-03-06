# bump micro_version to the next even number for each point release
# bump micro_version to the next odd number after each release
# keep interface_age in sync with micro_version, or set to 0 during development cycles
m4_define([graphene_major_version], [1])
m4_define([graphene_minor_version], [3])
m4_define([graphene_micro_version], [1])
m4_define([graphene_interface_age], [0])

m4_define([graphene_version], [graphene_major_version.graphene_minor_version.graphene_micro_version])

m4_define([graphene_release_status],
          [m4_if(m4_eval(graphene_micro_version % 2), [1], [git],
                 [m4_if(m4_eval(graphene_minor_version % 2), [1], [snapshot], [release])])])

m4_define([graphene_binary_age], [m4_eval(100 * graphene_minor_version + graphene_micro_version)])

m4_define([lt_current], [m4_eval(100 * graphene_minor_version + graphene_micro_version - graphene_interface_age)])
m4_define([lt_revision], [graphene_interface_age])
m4_define([lt_age], [m4_eval(graphene_binary_age - graphene_interface_age)])

m4_define([glib_req_version], [2.30.0])
m4_define([gi_req_version], [1.41.0])

AC_PREREQ([2.63])

AC_INIT([graphene],
        [graphene_version],
        [https://github.com/ebassi/graphene/issues],
        [graphene],
        [https://ebassi.github.io/graphene])

AC_CONFIG_HEADER([src/config.h])
AC_CONFIG_SRCDIR([src/graphene.h])
AC_CONFIG_AUX_DIR([build])
AC_CONFIG_MACRO_DIR([build/autotools])

# Honor aclocal flags
ACLOCAL="$ACLOCAL $ACLOCAL_FLAGS"

AM_INIT_AUTOMAKE([1.11 no-define foreign -Wno-portability dist-xz no-dist-gzip tar-ustar])

AM_SILENT_RULES([yes])

AM_MAINTAINER_MODE([enable])

AC_USE_SYSTEM_EXTENSIONS

dnl === Base compiler flags ===================================================

AC_PROG_SED

dnl Remove all optimization flags from CFLAGS; we are going to set -O3
dnl ourselves in BASE_CFLAGS using graphene_DEFAULT_COMPILER_FLAGS
changequote({,})
CFLAGS=`echo "$CFLAGS" | $SED -e 's/-O[s0-9]//g'`
changequote([,])

BASE_CFLAGS=""
graphene_DEFAULT_COMPILER_FLAGS

dnl === Libtool ===============================================================

LT_PREREQ([2.2.6])
LT_INIT([disable-static])
LT_LIB_M

dnl === Versioning ============================================================

GRAPHENE_MAJOR_VERSION=graphene_major_version
GRAPHENE_MINOR_VERSION=graphene_minor_version
GRAPHENE_MICRO_VERSION=graphene_micro_version
GRAPHENE_VERSION=graphene_version
GRAPHENE_API_VERSION=graphene_major_version.0
GRAPHENE_RELEASE_STATUS=graphene_release_status
AC_SUBST(GRAPHENE_MAJOR_VERSION)
AC_SUBST(GRAPHENE_MICRO_VERSION)
AC_SUBST(GRAPHENE_MINOR_VERSION)
AC_SUBST(GRAPHENE_VERSION)
AC_SUBST(GRAPHENE_API_VERSION)
AC_SUBST(GRAPHENE_RELEASE_STATUS)

GRAPHENE_LT_CURRENT=lt_current
GRAPHENE_LT_REVISION=lt_revision
GRAPHENE_LT_AGE=lt_age
GRAPHENE_LT_VERSION="$GRAPHENE_LT_CURRENT:$GRAPHENE_LT_REVISION:$GRAPHENE_LT_AGE"
GRAPHENE_LT_LDFLAGS="-version-info $GRAPHENE_LT_VERSION"

# built in SIMD backends
GRAPHENE_SIMD="scalar"

dnl === Dependencies ==========================================================

GRAPHENE_REQS=""

PKG_PROG_PKG_CONFIG

dnl === Platform checks =======================================================

platform_linux=no
platform_win32=no

AC_CANONICAL_HOST
AC_MSG_CHECKING([if building for some Win32 platform])
AS_CASE([$host],

        [*-*-mingw*|*-*-msys*],
        [
          GRAPHENE_LT_LDFLAGS="$GRAPHENE_LT_LDFLAGS -no-undefined"
          platform_win32=yes
          os_win32=yes
        ],

        [*-*-cygwin*],
        [
          GRAPHENE_LT_LDFLAGS="$GRAPHENE_LT_LDFLAGS -no-undefined"
          platform_win32=yes
          os_win32=no
        ],

        [*-*-linux*],
        [
          platform_linux=yes
        ],

        []
)
AC_MSG_RESULT([$platform_win32])

AM_CONDITIONAL(OS_LINUX, [test "$platform_linux" = "yes"])
AM_CONDITIONAL(OS_WIN32, [test "$platform_win32" = "yes"])

AC_CHECK_FUNCS([aligned_alloc posix_memalign memalign])

saved_LIBS="$LIBS"
LIBS=$LIBM
AC_CHECK_FUNCS([sincosf])
LIBS="$saved_LIBS"

AX_PTHREAD

AS_IF([test "x$os_win32" = "xyes"], [
  AC_COMPILE_IFELSE([AC_LANG_PROGRAM(
[[
#define _WIN32_WINNT 0x0600
#include <windows.h>
INIT_ONCE g_InitOnce = INIT_ONCE_STATIC_INIT;
BOOL CALLBACK InitHandleFunc (PINIT_ONCE i, PVOID arg, PVOID *ctx) { return TRUE; }
int main (void) {
  BOOL bInitStatus = InitOnceExecuteOnce (&g_InitOnce, InitHandleFunc, NULL, NULL);
  return 0;
}
]])],
                    [AC_DEFINE([HAVE_INIT_ONCE], [1], [Define if you have InitOnceExecuteOnce])],
                    [])
])

dnl === Fast paths ============================================================

# Check for GCC vectors
AC_ARG_ENABLE([gcc-vector],
              [AC_HELP_STRING([--disable-gcc-vector], [disable GCC vector fast paths])],
              [enable_gcc_vector=$enableval],
              [enable_gcc_vector=auto])

AC_CACHE_CHECK([whether to use GCC vector intrinsics],
               [graphene_cv_use_gcc_vector],
               [
                 graphene_cv_use_gcc_vector=no

                 AS_IF([test "x$enable_gcc_vector" = xno],
                       [graphene_cv_use_gcc_vector="no (disabled)"],
                       [
                 AC_COMPILE_IFELSE([AC_LANG_SOURCE([[
#if defined(__GNUC__)
# if __GNUC__ < 4 || (__GNUC__ == 4 && __GNUC_MINOR__ < 9)
#   error "GCC vector intrinsics are disabled on GCC prior to 4.9"
# elif defined(__arm__)
#   error "GCC vector intrinsics are disabled on ARM"
# endif
#else
# error "Need GCC for GCC vectors intrinsics"
#endif
typedef float simd4f __attribute__((vector_size(16)));
typedef int simd4i __attribute__((vector_size(16)));
int main () {
  simd4f s = { 1.f, 2.f, 3.f, 4.f };
  simd4i m = { 0,   1,   1,   3   };
  simd4f r = __builtin_shuffle (s, m);
  return 0;
}]])],  [graphene_cv_use_gcc_vector=yes])
                       ])
               ])

AS_IF([test "x$enable_gcc_vector" = xyes && test "x$graphene_cv_use_gcc_vector" = xno],
      [
        AC_MSG_ERROR([GNUC vector intrinsics not detected when explicitly enabled])
      ])

AS_IF([test "x$graphene_cv_use_gcc_vector" = xyes], [GRAPHENE_SIMD="gcc $GRAPHENE_SIMD"])

AM_CONDITIONAL(BUILD_WITH_GCC, [test "x$graphene_cv_use_gcc_vector" = xyes])

# Check for SSE2 intrinsics
AS_IF([test "x$SSE2_CFLAGS" = "x"],
      [
        AS_IF([test "x$SUNCC" = "xyes"],
              [
                # SSE2 is enabled by default in the Sun Studio 64-bit environment
                AS_IF([test "$AMD64_ABI" = "no"], [SSE2_CFLAGS="-xarch=sse2"])
              ],
              [
                SSE2_CFLAGS="-mfpmath=sse -msse -msse2"
              ])
      ])

# this block was nibbed from Pixman; I don't have Solaris nor SunCC, so I am
# assuming somebody has actually tested it, and it continues to work with
# Pixman.
AS_CASE([$host_os],

        [solaris*],
        [
          # When building 32-bit binaries, apply a mapfile to ensure that the
          # binaries aren't flagged as only able to run on MMX+SSE capable CPUs
          # since they check at runtime before using those instructions.
          # Not all linkers grok the mapfile format so we check for that first.
          AS_IF([test "$AMD64_ABI" = "no"],
                [
                  use_hwcap_mapfile=no
                  AC_MSG_CHECKING([whether to use a hardware capability map file])
                  hwcap_save_LDFLAGS="$LDFLAGS"
                  HWCAP_LDFLAGS='-Wl,-M,$(top_srcdir)/build/solaris-hwcap.mapfile'
                  LDFLAGS="$LDFLAGS -Wl,-M,build/solaris-hwcap.mapfile"
                  AC_LINK_IFELSE([AC_LANG_SOURCE([[int main() { return 0; }]])],
                                 [use_hwcap_mapfile=yes],
                                 [HWCAP_LDFLAGS=""])
                  LDFLAGS="$hwcap_save_LDFLAGS"
                  AC_MSG_RESULT($use_hwcap_mapfile)
                ])
          AS_IF([test "x$MMX_LDFLAGS" = "x"], [MMX_LDFLAGS="$HWCAP_LDFLAGS"])
          AS_IF([test "x$SSE2_LDFLAGS" = "x"], [SSE2_LDFLAGS="$HWCAP_LDFLAGS"])
        ]
)

AC_ARG_ENABLE([sse2],
              [AC_HELP_STRING([--disable-sse2], [disable SSE2 fast paths])],
              [enable_sse2=$enableval],
              [enable_sse2=auto])

AC_CACHE_CHECK([whether to use SSE2 intrinsics],
               [graphene_cv_use_sse2],
               [
                 graphene_cv_use_sse2=no

                 AS_IF([test "x$enable_sse2" = xno],
                       [graphene_cv_use_sse2="no (disabled)"],
                       [
                         saved_CFLAGS="$CFLAGS"
                         CFLAGS="$SSE2_CFLAGS $CFLAGS"

                         AC_COMPILE_IFELSE([AC_LANG_SOURCE([[
#if defined(__GNUC__) && (__GNUC__ < 4 || (__GNUC__ == 4 && __GNUC_MINOR__ < 2))
# if !defined(__amd64__) && !defined(__x86_64__)
#   error "Need GCC >= 4.2 for SSE2 intrinsics on x86"
# endif
#endif
#if defined(__SSE__) || (_M_IX86_FP > 0) || (_M_X64 > 0)
# include <mmintrin.h>
# include <xmmintrin.h>
# include <emmintrin.h>
#else
# error "No SSE intrinsics available"
#endif
int main () {
    __m128i a = _mm_set1_epi32 (0), b = _mm_set1_epi32 (0), c;
    c = _mm_xor_si128 (a, b);
    return 0;
}]])], [graphene_cv_use_sse2=yes])
                        CFLAGS="$saved_CFLAGS"
                      ])
               ])

AS_IF([test "x$enable_sse2" = xyes && test "x$graphene_cv_use_sse2" = xno],
      [
        AC_MSG_ERROR([SSE2 intrinsics not detected when explicitly enabled])
      ])

AS_IF([test "x$graphene_cv_use_sse2" = xyes], [GRAPHENE_SIMD="sse2 $GRAPHENE_SIMD"])

AM_CONDITIONAL(BUILD_WITH_SSE2, [test "x$graphene_cv_use_sse2" = xyes])

# Check for SSE4.1
AS_IF([test "x$graphene_cv_use_sse2" = xyes], [
  AS_IF([test "x$SSE41_CFLAGS" = "x"], [ SSE41_CFLAGS="-msse4.1" ])

  AC_CACHE_CHECK([whether to use SSE4.1 intrinsics],[graphene_cv_use_sse41], [
  graphene_cv_use_sse41=no

  saved_CFLAGS="$CFLAGS"
  CFLAGS="$SSE2_CFLAGS $SSE41_CFLAGS $CFLAGS"

  AC_COMPILE_IFELSE([AC_LANG_SOURCE([[
#include <mmintrin.h>
#include <xmmintrin.h>
#if defined(_M_IX86_FP)
# if _M_IX86_FP >=2
#  define USE_SSE4_1
# endif
#elif defined(__SSE4_1__)
# define USE_SSE4_1
#endif
#if defined(USE_SSE4_1)
# include <smmintrin.h>
#endif
int main () {
    __m128 a = _mm_setzero_ps(), b = _mm_setzero_ps(), c;
    c = _mm_dp_ps(a, b, 0x71);
    return 0;
}]])], [graphene_cv_use_sse41=yes])

  CFLAGS="$saved_CFLAGS"
  ])
])

AS_IF([test "x$graphene_cv_use_sse2" = xyes -a "x$graphene_cv_use_sse41" = xyes], [
  SSE2_CFLAGS="$SSE2_CFLAGS $SSE41_CFLAGS"
])

# Check for ARM NEON instructions
AS_IF([test "x$NEON_CFLAGS" = "x"], [NEON_CFLAGS="-mfpu=neon"])

# the Android toolchain needs some further handholding
AC_MSG_CHECKING([if we are on Android])
AS_CASE([$host_os],

        [*android*],
        [
          AC_MSG_RESULT([yes])
          NEON_CFLAGS="$NEON_CFLAGS -mfloat-abi=softfp"
        ],
        
        [AC_MSG_RESULT([no])]
)

AC_ARG_ENABLE([arm-neon],
              [AC_HELP_STRING([--disable-arm-neon], [disable ARM NEON fast paths])],
              [enable_arm_neon=$enableval],
              [enable_arm_neon=auto])

AC_CACHE_CHECK([whether to use ARM NEON intrinsics],
               [graphene_cv_use_arm_neon],
               [
                 graphene_cv_use_arm_neon=no

                 AS_IF([test "x$enable_arm_neon" = xno],
                       [graphene_cv_use_arm_neon="no (disabled)"],
                       [
                         saved_CFLAGS="$CFLAGS"
                         CFLAGS="$NEON_CFLAGS $CFLAGS"

                         AC_COMPILE_IFELSE([AC_LANG_SOURCE([[
#ifndef __ARM_EABI__
#error "EABI is required (to be sure that calling conventions are compatible)"
#endif
#ifndef __ARM_NEON__
#error "No ARM NEON instructions available"
#endif
#include <arm_neon.h>
int main () {
    const float32_t __v[4] = { 1, 2, 3, 4 }; \
    const unsigned int __umask[4] = { \
      0x80000000, \
      0x80000000, \
      0x80000000, \
      0x80000000 \
    }; \
    const uint32x4_t __mask = vld1q_u32 (__umask); \
    float32x4_t s = vld1q_f32 (__v); \
    float32x4_t c = vreinterpretq_f32_u32 (veorq_u32 (vreinterpretq_u32_f32 (s), __mask)); \
    return 0;
}]])], [graphene_cv_use_arm_neon=yes])

                         CFLAGS="$saved_CFLAGS"
                       ])
               ])

AS_IF([test "x$enable_arm_neon" = xyes && test "x$graphene_cvs_use_arm_neon" = xno],
      [
        AC_MSG_ERROR([ARM NEON intrinsics not detected when explicitly enabled])
      ])

AS_IF([test "x$graphene_cv_use_arm_neon" = xyes], [GRAPHENE_SIMD="neon $GRAPHENE_SIMD"])

AM_CONDITIONAL(BUILD_WITH_ARM_NEON, [test "x$graphene_cv_use_arm_neon" = xyes])

# Check for -Bsymbolic-functions to avoid intra-library PLT jumps
AC_ARG_ENABLE([Bsymbolic],
              [AS_HELP_STRING([--disable-Bsymbolic], [Avoid linking with -Bsymbolic])],
              [],
              [
                saved_LDFLAGS="${LDFLAGS}"

                AC_MSG_CHECKING([for -Bsymbolic-functions linker flag])
                LDFLAGS=-Wl,-Bsymbolic-functions
                AC_TRY_LINK([], [int main (void) { return 0; }],
                            [ enable_Bsymbolic=yes ],
                            [ enable_Bsymbolic=no  ])
                AC_MSG_RESULT([$enable_Bsymbolic])

                LDFLAGS="${saved_LDFLAGS}"
              ])

AS_IF([test "x$enable_Bsymbolic" = xyes], [GRAPHENE_LINK_FLAGS=-Wl[,]-Bsymbolic-functions])

dnl === Visibility ============================================================

GRAPHENE_VISIBILITY_CFLAGS=""
AS_CASE(["$host"],

        [*-*-mingw*],
        [
          dnl on mingw32 we do -fvisibility=hidden and __declspec(dllexport)
          AC_DEFINE([_GRAPHENE_PUBLIC],
                    [__attribute__((visibility("default"))) __declspec(dllexport) extern],
                    [defines how to decorate public symbols while building])
          GRAPHENE_VISIBILITY_CFLAGS="-fvisibility=hidden"
        ],

        [
          dnl on other compilers, check if we can do -fvisibility=hidden
          SAVED_CFLAGS="${CFLAGS}"
          CFLAGS="-fvisibility=hidden"
          AC_MSG_CHECKING([for -fvisibility=hidden compiler flag])
          AC_TRY_COMPILE([], [int main (void) { return 0; }],
                         [
                           AC_MSG_RESULT(yes)
                           enable_fvisibility_hidden=yes
                         ],
                         [
                           AC_MSG_RESULT(no)
                           enable_fvisibility_hidden=no
                         ])
          CFLAGS="${SAVED_CFLAGS}"

          AS_IF([test "${enable_fvisibility_hidden}" = "yes"],
                [
                  AC_DEFINE([_GRAPHENE_PUBLIC],
                            [__attribute__((visibility("default"))) extern],
                            [defines how to decorate public symbols while building])
                  GRAPHENE_VISIBILITY_CFLAGS="-fvisibility=hidden"
                ])
        ]
)

dnl === Enable debug level ====================================================

m4_define([debug_default], [m4_if(m4_eval(graphene_minor_version % 2), [1], [yes], [minimum])])

AC_ARG_ENABLE([debug],
              [AS_HELP_STRING([--enable-debug=@<:@no/minimum/yes@:>@],
                              [turn on debugging @<:@default=]debug_default[@:>@])],
              [],
              [enable_debug=debug_default])

AS_CASE([$enable_debug],

        [yes],
        [
          BASE_CFLAGS="-g $BASE_CFLAGS"
          GRAPHENE_DEBUG_CFLAGS="-DGRAPHENE_ENABLE_DEBUG"
        ],

        [minimum],
        [
          GRAPHENE_DEBUG_CFLAGS="-DGRAPHENE_ENABLE_DEBUG -DG_DISABLE_CAST_CHECKS"
        ],

        [no],
        [
          GRAPHENE_DEBUG_CFLAGS="-DG_DISABLE_ASSERT -DG_DISABLE_CHECKS -DG_DISABLE_CAST_CHECKS"
        ],

        [AC_MSG_ERROR([Unknown argument to --enable-debug])]
)

AC_SUBST(GRAPHENE_DEBUG_CFLAGS)

dnl === GLib ==================================================================

PKG_CHECK_MODULES(GLIB,
                  [glib-2.0 >= glib_req_version],
                  [has_glib=yes],
                  [has_glib=no])

dnl === GObject types ancillary libary ========================================

AC_ARG_ENABLE([gobject-types],
              [AS_HELP_STRING([--disable-gobject-types],
                              [Disable building GObject types])],
              [enable_gobject=$enableval],
              [enable_gobject=yes])

AS_IF([test "x$enable_gobject" = xyes],
      [
        PKG_CHECK_MODULES(GRAPHENE_GOBJECT, [gobject-2.0],
                          [build_gobject=yes],
                          [build_gobject="disabled (missing GObject)"])
        AS_IF([test "x$build_gobject" = xyes], [GRAPHENE_REQS="$GRAPHENE_REQS gobject-2.0"])
      ],
      [
        build_gobject=disabled
      ])

AM_CONDITIONAL(BUILD_GOBJECT, [test "x$build_gobject" = xyes])

dnl === GObject introspection =================================================

AS_IF([test "x$build_gobject" = xyes],
      [
        GOBJECT_INTROSPECTION_CHECK([gi_req_version])
      ],
      [
        AM_CONDITIONAL(HAVE_INTROSPECTION, [false])
        enable_introspection=disabled
      ])

dnl === Test suite (require GLib) =============================================

AC_ARG_ENABLE([tests],
              [AS_HELP_STRING([--enable-tests], [Enable test suite])],
              [enable_tests=$enableval],
              [enable_tests=check])

GLIB_TESTS

AS_CASE([$enable_tests],
        [check],
        [
          AS_IF([test "x$has_glib" = xyes],
                [build_tests=yes],
                [build_tests="disabled (missing GLib)"])
        ],
        
        [yes],
        [
          AS_IF([test "x$has_glib" = xyes],
                [build_tests=yes],
                [AC_MSG_ERROR([Cannot build the test suite without GLib])])
        ],

        [no],
        [ build_tests=disabled ])

AM_CONDITIONAL(BUILD_TESTS, [test "x$build_tests" = xyes])

dnl === Benchmarks (require GLib) =============================================

AC_ARG_ENABLE([benchmarks],
              [AS_HELP_STRING([--enable-benchmarks], [Enable benchmarks])],
              [enable_benchmarks=$enableval],
              [enable_benchmarks=check])

AS_CASE([$enable_benchmarks],
        [check],
        [
          AS_IF([test "x$has_glib" = xyes],
                [build_benchmarks=yes],
                [build_benchmarks="disabled (missing GLib)"])
        ],

        [yes],
        [
          AS_IF([test "x$has_glib" = xyes],
                [build_benchmarks=yes],
                [AC_MSG_ERROR([Cannot build the benchmarks suite without GLib])])
        ],

        [no],
        [ build_benchmarks=disabled ])

AM_CONDITIONAL(BUILD_BENCHMARKS, [test "x$build_benchmarks" = xyes])

dnl === Code coverage =========================================================

AC_ARG_ENABLE([code-coverage],
              [AS_HELP_STRING([--enable-code-coverage], [Enable test suite code coverage])],
              [use_gcov=$enableval],
              [use_gcov=no])

dnl Disable code coverage if tests were disabled
AS_IF([test "x$build_tests" = disabled ], [ use_gcov=disabled ])

AS_IF([test "x$use_gcov" = "xyes"],
      [
        dnl we need a test suite
        AS_IF([test "x$build_tests" = xno],
              [AC_MSG_ERROR([Cannot enable code coverage when the test suite is disabled])])

        dnl we need gcc
        AS_IF([test "$GCC" != "yes"], [AC_MSG_ERROR([GCC is required for --enable-code-coverage])])

        dnl Check if ccache is being used
        AC_CHECK_PROG(SHTOOL, shtool, shtool)
        AS_CASE([`$SHTOOL path $CC`],
                [*ccache*], [gcc_ccache=yes],
                [gcc_ccache=no])

        if test "$gcc_ccache" = "yes" && (test -z "$CCACHE_DISABLE" || test "$CCACHE_DISABLE" != "1"); then
          AC_MSG_ERROR([ccache must be disabled when --enable-code-coverage option is used. You can disable ccache by setting environment variable CCACHE_DISABLE=1.])
        fi

        ltp_version_list="1.6 1.7 1.8 1.9 1.10"
        AC_CHECK_PROG(LTP, lcov, lcov)
        AC_CHECK_PROG(LTP_GENHTML, genhtml, genhtml)

        if test "$LTP"; then
          AC_CACHE_CHECK([for ltp version], graphene_cv_ltp_version,
                         [
                           graphene_cv_ltp_version=invalid
                           ltp_version=`$LTP -v 2>/dev/null | $SED -e 's/^.* //'`
                           for ltp_check_version in $ltp_version_list; do
                             if test "$ltp_version" = "$ltp_check_version"; then
                               graphene_cv_ltp_version="$ltp_check_version (ok)"
                             fi
                           done
                         ])
        else
          ltp_msg="To enable code coverage reporting you must have one of the following LTP versions installed: $ltp_version_list"
          AC_MSG_ERROR([$ltp_msg])
        fi

        case $graphene_cv_ltp_version in
          ""|invalid[)]
          ltp_msg="You must have one of the following versions of LTP: $ltp_version_list (found: $ltp_version)."
          AC_MSG_ERROR([$ltp_msg])
          LTP="exit 0;"
          ;;
        esac

        if test -z "$LTP_GENHTML"; then
          AC_MSG_ERROR([Could not find genhtml from the LTP package])
        fi

        AC_DEFINE(HAVE_GCOV, 1, [Whether you have gcov])

        dnl Remove all compiler optimization flags
        changequote({,})
        BASE_CFLAGS=`echo "$BASE_CFLAGS" | $SED -e 's/-O[s0-9]//g'`
        changequote([,])

        dnl Define the special gcc flags
        GRAPHENE_GCOV_CFLAGS="-O0 -fprofile-arcs -ftest-coverage"
        GRAPHENE_GCOV_LDADD="-lgcov"
      ])

AM_CONDITIONAL(ENABLE_GCOV, [test "x$use_gcov" = xyes])

dnl === API Reference =========================================================

GLIB_PREFIX="`$PKG_CONFIG --variable=prefix glib-2.0`"
AC_SUBST(GLIB_PREFIX)

# gtkdocize greps for ^GTK_DOC_CHECK, so we need to put it on its own line
m4_ifdef([GTK_DOC_CHECK], [
GTK_DOC_CHECK([1.20], [--flavour no-tmpl])
],[
enable_gtk_doc="disabled (no gtk-doc)"
AM_CONDITIONAL([ENABLE_GTK_DOC], [false])
])

dnl === Output ================================================================

GRAPHENE_CFLAGS="$BASE_CFLAGS $GRAPHENE_VISIBILITY_CFLAGS $GRAPHENE_GCOV_CFLAGS $GRAPHENE_GOBJECT_CFLAGS $PTHREAD_CFLAGS"
GRAPHENE_LIBS="$LIBM $GRAPHENE_GCOV_LDADD $GRAPHENE_GOBJECT_LIBS $PTHREAD_LIBS"
GRAPHENE_LDFLAGS="$GRAPHENE_LT_LDFLAGS $GRAPHENE_LINK_FLAGS $SSE2_LDFLAGS $MMX_LDFLAGS"
AC_SUBST(GRAPHENE_CFLAGS)
AC_SUBST(GRAPHENE_LIBS)
AC_SUBST(GRAPHENE_LDFLAGS)
AC_SUBST(GRAPHENE_REQS)
AC_SUBST(GRAPHENE_SIMD)
AC_SUBST(SSE2_CFLAGS)
AC_SUBST(SSE2_LDFLAGS)
AC_SUBST(NEON_CFLAGS)
AC_SUBST(GLIB_CFLAGS)
AC_SUBST(GLIB_LIBS)
AC_SUBST(PTHREAD_CFLAGS)
AC_SUBST(PTHREAD_LIBS)

AC_CONFIG_COMMANDS([src/graphene-config.h],
[
  outfile=src/graphene-config.h-tmp
  cat > $outfile <<\_______EOF
/* graphene-config.h
 *
 * This is a generated file. Please modify 'configure.ac'.
 */

#ifndef __GRAPHENE_CONFIG_H__
#define __GRAPHENE_CONFIG_H__

#ifdef __cplusplus
extern "C" {
#endif

#ifndef GRAPHENE_SIMD_BENCHMARK
_______EOF

        if test "x$graphene_has_sse2" = xyes; then
          echo '# if defined(__SSE__) || (_M_IX86_FP > 0) || (_M_X64 > 0)' >> $outfile
          echo '#  define GRAPHENE_HAS_SSE 1' >> $outfile
          echo '# endif' >> $outfile
        fi
        if test "x$graphene_has_arm_neon" = xyes; then
          echo '# if defined(__ARM_NEON__)' >> $outfile
          echo '#  define GRAPHENE_HAS_ARM_NEON 1' >> $outfile
          echo '# endif' >> $outfile
        fi
        if test "x$graphene_has_gcc_vector" = xyes; then
          echo '# if defined(__GNUC__) && (__GNUC__ >= 4 && __GNUC_MINOR__ >= 9) && !defined(__arm__)' >> $outfile
          echo '#  define GRAPHENE_HAS_GCC 1' >> $outfile
          echo '# endif' >> $outfile
        fi

        cat >> $outfile <<_______EOF
# define GRAPHENE_HAS_SCALAR 1
#endif /* GRAPHENE_SIMD_BENCHMARK */

#if defined(GRAPHENE_HAS_SSE)
# define GRAPHENE_USE_SSE
# define GRAPHENE_SIMD_S "sse"
#elif defined(GRAPHENE_HAS_ARM_NEON)
# define GRAPHENE_USE_ARM_NEON
# define GRAPHENE_SIMD_S "neon"
#elif defined(GRAPHENE_HAS_GCC)
# define GRAPHENE_USE_GCC
# define GRAPHENE_SIMD_S "gcc"
#elif defined(GRAPHENE_HAS_SCALAR)
# define GRAPHENE_USE_SCALAR
# define GRAPHENE_SIMD_S "scalar"
#else
# error "Unsupported platform."
#endif

#ifndef __GI_SCANNER__
# if defined(GRAPHENE_USE_SSE)
#  include <xmmintrin.h>
#  include <emmintrin.h>
#  if defined(_M_IX86_FP)
#   if _M_IX86_FP >= 2
#    define GRAPHENE_USE_SSE4_1
#   endif
#  elif defined(__SSE4_1__)
#   define GRAPHENE_USE_SSE4_1
#  endif
#  if defined(GRAPHENE_USE_SSE4_1)
#   include <smmintrin.h>
#  endif
typedef __m128 graphene_simd4f_t;
# elif defined(GRAPHENE_USE_ARM_NEON)
#  include <arm_neon.h>
typedef float32x4_t graphene_simd4f_t;
# elif defined(GRAPHENE_USE_GCC)
typedef float graphene_simd4f_t __attribute__((vector_size(16)));
# elif defined(GRAPHENE_USE_SCALAR)
typedef struct {
  /*< private >*/
  float x, y, z, w;
} graphene_simd4f_t;
# else
#  error "Unsupported platform."
# endif
#else /* __GI_SCANNER__ */
/* The gobject-introspection scanner has issues parsing the
 * system headers with SIMD built-ins, so we fall back to
 * scalars; it does not really matter, as we wrap them in
 * our public API, and introspection cannot use the SIMD API
 * directly anyway.
 */
typedef struct {
  /*< private >*/
  float x, y, z, w;
} graphene_simd4f_t;
#endif /* __GI_SCANNER__ */

typedef struct {
  /*< private >*/
  graphene_simd4f_t x, y, z, w;
} graphene_simd4x4f_t;

#ifdef __cplusplus
}
#endif

#endif /* __GRAPHENE_CONFIG_H__ */
_______EOF
        if cmp -s $outfile src/graphene-config.h ; then
          AC_MSG_NOTICE([src/graphene-config.h is unchanged])
          rm -f $outfile
        else
          mv $outfile src/graphene-config.h
        fi
], [
graphene_has_sse2="$graphene_cv_use_sse2"
graphene_has_arm_neon="$graphene_cv_use_arm_neon"
graphene_has_gcc_vector="$graphene_cv_use_gcc_vector"
])

AC_CONFIG_FILES([
        Makefile

        src/Makefile
        src/tests/Makefile
        src/bench/Makefile

        src/graphene-$GRAPHENE_API_VERSION.pc:src/graphene.pc.in
        src/graphene-gobject-$GRAPHENE_API_VERSION.pc:src/graphene.pc.in

        src/graphene-version.h

        doc/Makefile
])

AC_OUTPUT

dnl === Summary ===============================================================

AC_MSG_NOTICE([

Graphene - $VERSION (${GRAPHENE_RELEASE_STATUS})

  • Prefix: ${prefix}

  • Documentation: ${enable_gtk_doc}
  • Debug level: ${enable_debug}
  • Test suite: ${build_tests}
  • Code coverage: ${use_gcov}
  • Benchmarks: ${build_benchmarks}
  • GObject types: ${build_gobject}
  • GObject introspection: ${enable_introspection}

  • Available optimizations:
    • GCC vectors: ${graphene_cv_use_gcc_vector}
    • SSE2 intrinsics: ${graphene_cv_use_sse2}
     • SSE4.1 intrinsics: ${graphene_cv_use_sse41}
    • ARM NEON intrinsics: ${graphene_cv_use_arm_neon}
])
