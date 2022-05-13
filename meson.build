project('crc32c', 'cpp', default_options: ['c_std=c11', 'cpp_std=c++11'])

cc = meson.get_compiler('cpp')
cpp_args = ['-fno-exceptions', '-fno-rtti']

conf = configuration_data()
conf.set10('CRC32C_TESTS_BUILT_WITH_GLOG', false)
conf.set10('BYTE_ORDER_BIG_ENDIAN', build_machine.endian() == 'big')

# Check for __builtin_prefetch support in the compiler.
conf.set10('HAVE_BUILTIN_PREFETCH', cc.compiles('''
int main() {
  char data = 0;
  const char* address = &data;
  __builtin_prefetch(address, 0, 0);
  return 0;
}
'''))

# Check for _mm_prefetch support in the compiler.
conf.set10('HAVE_MM_PREFETCH', cc.compiles('''
#include <xmmintrin.h>

int main() {
  char data = 0;
  const char* address = &data;
  _mm_prefetch(address, _MM_HINT_NTA);
  return 0;
}
'''))

# Check for SSE4.2 support in the compiler.
sse42_args = ['-msse4.2']
have_sse42 = cc.compiles('''
#include <cpuid.h>
#include <nmmintrin.h>

int main() {
  _mm_crc32_u8(0, 0); _mm_crc32_u32(0, 0);
#if defined(_M_X64) || defined(__x86_64__)
   _mm_crc32_u64(0, 0);
#endif // defined(_M_X64) || defined(__x86_64__)
  return 0;
}
''',
args: sse42_args)

conf.set10('HAVE_SSE42', have_sse42)
cpp_args += have_sse42 ? sse42_args : []


# Check for ARMv8 w/ CRC and CRYPTO extensions support in the compiler.
arm64_args = ['-march=armv8-a+crc+crypto']
have_arm64 = cc.compiles('''
#include <arm_acle.h>
#include <arm_neon.h>

int main() {
  __crc32cb(0, 0); __crc32ch(0, 0); __crc32cw(0, 0); __crc32cd(0, 0);
  vmull_p64(0, 0);
  return 0;
}
''',
args: arm64_args)

conf.set10('HAVE_ARM64_CRC32C', have_arm64)
cpp_args += have_arm64 ? arm64_args : []

# Check for strong getauxval() support in the system headers.
conf.set10('HAVE_STRONG_GETAUXVAL', cc.compiles('''
#include <arm_acle.h>
#include <arm_neon.h>
#include <sys/auxv.h>

int main() {
  getauxval(AT_HWCAP);
  return 0;
}
'''))

# Check for weak getauxval() support in the compiler.
conf.set10('HAVE_WEAK_GETAUXVAL', cc.compiles('''
unsigned long getauxval(unsigned long type) __attribute__((weak));
#define AT_HWCAP 16

int main() {
  getauxval(AT_HWCAP);
  return 0;
}
'''))

subdir('include/crc32c')
inc = include_directories('include')

# ARM64 CRC32C code is built separately, so we don't accidentally compile
# unsupported instructions into code that gets run without ARM32 support.
crc32c_arm64 = static_library('crc32c_arm64',
  'src/crc32c_arm64.cc',
  config_h,
  include_directories: inc,
  cpp_args: cpp_args)

# SSE4.2 code is built separately, so we don't accidentally compile unsupported
# instructions into code that gets run without SSE4.2 support.
crc32c_sse42 = static_library('crc32c_sse42',
  'src/crc32c_sse42.cc',
  config_h,
  include_directories: inc,
  cpp_args: cpp_args)

crc32c = library('crc32c',
  'src/crc32c_portable.cc',
  'src/crc32c.cc',
  config_h,
  include_directories: inc,
  link_whole: [crc32c_arm64, crc32c_sse42])

crc32c_dep = declare_dependency(
  include_directories: inc,
  link_with: crc32c)
