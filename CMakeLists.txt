cmake_minimum_required(VERSION 3.12)
project(libgpg-error
		VERSION 1.41
		LANGUAGES C)

add_compile_definitions(HAVE_CONFIG_H
PKGDATADIR="${CMAKE_INSTALL_PREFIX}/share/${PROJECT_NAME}"
LOCALEDIR="${CMAKE_INSTALL_PREFIX}/share/locale")

if(WIN32)
	add_compile_definitions(DLL_EXPORT)
endif()

set(PACKAGE \"${PROJECT_NAME}\")
set(PACKAGE_BUGREPORT \"https://bugs.gnupg.org\")
set(PACKAGE_NAME \"${PROJECT_NAME}\")
set(PACKAGE_STRING \"${PROJECT_NAME}\ ${PROJECT_VERSION}\")
set(PACKAGE_TARNAME \"${PROJECT_NAME}\")
set(PACKAGE_URL \"\")
set(PACKAGE_VERSION \"${PROJECT_VERSION}\")
set(VERSION \"${PROJECT_VERSION}\")
set(VERSION_HEX 0x012900) #0x%02x%02x00 -> 0x'major''minor'00

include(UtilityFunctions)
include(CheckCSourceCompiles)
include(CheckLibraryExists)
include(visibility)
include(LargeFiles)
include(fork)
include(TimeWithSysTime)

find_package(Iconv)
find_package(Intl)
find_program(GAWK_EXE gawk REQUIRED)
find_program(GREP_EXE grep REQUIRED)

file(TO_NATIVE_PATH ${GAWK_EXE} GAWK_EXE)
file(TO_NATIVE_PATH ${GREP_EXE} GREP_EXE)

set(BUILD_REVISION \"\")
set(BUILD_TIMESTAMP \"\")

if(Iconv_FOUND)
	set(HAVE_ICONV 1)
endif()

if(Intl_FOUND)
	set(ENABLE_NLS 1)
	set(HAVE_GETTEXT 1)
	set(HAVE_DCGETTEXT 1)
endif()


if(WIN32)
	set(HAVE_VISIBILITY 0)
	set(HAVE_W32_SYSTEM 1)
	set(HAVE_W64_SYSTEM 1)
	set(MKDIR_TAKES_ONE_ARG 1)
endif()

if(WIN32)
	set(USE_WINDOWS_THREADS 1)
	set(HOST_TRIPLET_STRING \"x86_64-windows\")
else()
	set(USE_POSIX_THREADS 1)
	set(HOST_TRIPLET_STRING \"x86_64-pc-linux-gnu\")
endif()

if(HAVE_VISIBILITY)
	set(GPGRT_USE_VISIBILITY 1)
endif()

check_c_headers(direct.h dlfcn.h inttypes.h locale.h memory.h poll.h
pwd.h signal.h stdint.h stdlib.h strings.h string.h sys/select.h sys/file.h
sys/stat.h sys/time.h sys/types.h unistd.h vfork.h pthread.h langinfo.h)

check_functions(flockfile fork getpwnam getpwnam_r getpwuid getpwuid_r
getrlimit memrchr mmap rand setenv stat stpcpy
strerror_r strlwr vasprintf vfork)

check_functions_declarations(strerror_r)

check_library_exists(readline rl_completion_matches "" HAVE_LIBREADLINE)

check_types(intmax_t "long double" "long long int" "unsigned long long int" ptrdiff_t uintmax_t
int long "long long" time_t "void *" pid_t off_t)

set(REPLACEMENT_FOR_OFF_T "long\ long")

if(NOT HAVE_PID_T)
	set(pid_t int)
endif()

if(NOT HAVE_INTMAX_T)
	set(intmax_t "long\ long")
endif()

if(NOT HAVE_UINTMAX_T)
	set(uintmax_t "unsigned\ long\ long")
endif()

set(CMAKE_EXTRA_INCLUDE_FILES "pthread.h")
check_types(pthread_rwlock_t pthread_mutex_t)
unset(CMAKE_EXTRA_INCLUDE_FILES)

if(HAVE_PTHREAD_RWLOCK_T)
	set(HAVE_PTHREAD_RWLOCK 1)
endif()

check_c_source_compiles("
struct { int a; } foo __attribute__ ((aligned (16)));
int main(){return 0;}"
HAVE_GCC_ATTRIBUTE_ALIGNED)

check_c_source_compiles("
#include <langinfo.h>
int
main ()
{
char* cs = nl_langinfo(THOUSEP); return !cs;
  return 0;
}"
HAVE_LANGINFO_THOUSEP)

check_c_source_compiles("
#include <pthread.h>
int
main ()
{
	int x = (int)PTHREAD_MUTEX_RECURSIVE;
	return !x;
}"
HAVE_PTHREAD_MUTEX_RECURSIVE)

check_c_source_runs("
#include <string.h>
#include <ctype.h>
extern char *strerror_r ();
int
main ()
{
	char buf[100];
	char x = *strerror_r (0, buf, sizeof buf);
	return ! isalpha (x);
}"
STRERROR_R_CHAR_P)

configure_file(libgpg-error-config.h.in config.h)
if(WIN32)
	if(NOT HAVE_SYS_FILE_H)
		configure_file(sys_file.h.in sys/file.h)
	endif()
endif()

include_directories(${CMAKE_CURRENT_BINARY_DIR} src)

if(MSVC)
add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/mkerrcodes.h
			COMMAND ${GAWK_EXE} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes1.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/errnos.in > ${CMAKE_CURRENT_BINARY_DIR}/_mkerrcodes.h
			COMMAND ${CMAKE_C_COMPILER} /EP ${CMAKE_CURRENT_BINARY_DIR}/_mkerrcodes.h | ${GREP_EXE} GPG_ERR_ |
			${GAWK_EXE} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes.awk > ${CMAKE_CURRENT_BINARY_DIR}/mkerrcodes.h
			COMMAND ${CMAKE_COMMAND} -E rm ${CMAKE_CURRENT_BINARY_DIR}/_mkerrcodes.h
			DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes1.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/errnos.in ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes.awk)
else()
add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/mkerrcodes.h
			COMMAND ${GAWK_EXE} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes1.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/errnos.in > ${CMAKE_CURRENT_BINARY_DIR}/_mkerrcodes.h
			COMMAND ${CMAKE_C_COMPILER} -E -P ${CMAKE_CURRENT_BINARY_DIR}/_mkerrcodes.h | ${GREP_EXE} GPG_ERR_ |
			${GAWK_EXE} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes.awk > ${CMAKE_CURRENT_BINARY_DIR}/mkerrcodes.h
			COMMAND ${CMAKE_COMMAND} -E rm ${CMAKE_CURRENT_BINARY_DIR}/_mkerrcodes.h
			DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes1.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/errnos.in ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes.awk)
endif()

add_custom_target(gen_mkerrcodes DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/mkerrcodes.h)
add_executable(mkerrcodes src/mkerrcodes.c)
add_dependencies(mkerrcodes gen_mkerrcodes)

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/code-from-errno.h
			COMMAND mkerrcodes | ${GAWK_EXE} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes2.awk > ${CMAKE_CURRENT_BINARY_DIR}/code-from-errno.h
			DEPENDS mkerrcodes ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes2.awk)

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/code-to-errno.h
			COMMAND ${GAWK_EXE} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrnos.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/errnos.in > ${CMAKE_CURRENT_BINARY_DIR}/code-to-errno.h
			DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrnos.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/errnos.in)

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/err-sources-sym.h
			COMMAND ${GAWK_EXE} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk -v textidx=2 -v nogettext=1
			${CMAKE_CURRENT_SOURCE_DIR}/src/err-sources.h.in > ${CMAKE_CURRENT_BINARY_DIR}/err-sources-sym.h
			DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/err-sources.h.in)

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/err-sources.h
			COMMAND ${GAWK_EXE} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk -v textidx=3
			${CMAKE_CURRENT_SOURCE_DIR}/src/err-sources.h.in > ${CMAKE_CURRENT_BINARY_DIR}/err-sources.h
			DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/err-sources.h.in)

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/err-codes-sym.h
			COMMAND ${GAWK_EXE} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk -v textidx=2 -v nogettext=1
			${CMAKE_CURRENT_SOURCE_DIR}/src/err-codes.h.in > ${CMAKE_CURRENT_BINARY_DIR}/err-codes-sym.h
			DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/err-codes.h.in)

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/err-codes.h
			COMMAND ${GAWK_EXE} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk -v textidx=3
			${CMAKE_CURRENT_SOURCE_DIR}/src/err-codes.h.in > ${CMAKE_CURRENT_BINARY_DIR}/err-codes.h
			DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/err-codes.h.in)

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/errnos-sym.h
			COMMAND ${GAWK_EXE} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk -v textidx=2 -v nogettext=1 -v prefix=GPG_ERR_ -v pkg_namespace=errnos_
			${CMAKE_CURRENT_SOURCE_DIR}/src/errnos.in > ${CMAKE_CURRENT_BINARY_DIR}/errnos-sym.h
			DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/errnos.in)

add_executable(mkheader-gpgrt src/mkheader.c)
if(NOT WIN32)
	add_executable(gen-lock src/gen-posix-lock-obj.c)
endif()

if(WIN32)
add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/lock-obj-pub.native.h
			COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/src/syscfg/lock-obj-pub.mingw32.h ${CMAKE_CURRENT_BINARY_DIR}/lock-obj-pub.native.h
			DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/syscfg/lock-obj-pub.mingw32.h)
else()
add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/lock-obj-pub.native.h
			COMMAND gen-lock > ${CMAKE_CURRENT_BINARY_DIR}/lock-obj-pub.native.h
			DEPENDS gen-lock)
endif()

add_custom_command(OUTPUT ${CMAKE_BINARY_DIR}/gpg-error.h ${CMAKE_BINARY_DIR}/gpgrt.h
			COMMAND mkheader-gpgrt ${HOST_TRIPLET_STRING} ${CMAKE_CURRENT_SOURCE_DIR}/src/gpg-error.h.in
			${CMAKE_CURRENT_BINARY_DIR}/config.h ${PROJECT_VERSION} ${VERSION_HEX} > ${CMAKE_BINARY_DIR}/gpg-error.h
			COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/gpg-error.h ${CMAKE_BINARY_DIR}/gpgrt.h
			DEPENDS mkheader-gpgrt
			${CMAKE_CURRENT_SOURCE_DIR}/src/gpg-error.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h ${CMAKE_CURRENT_BINARY_DIR}/lock-obj-pub.native.h
			${CMAKE_CURRENT_SOURCE_DIR}/src/err-sources.h.in ${CMAKE_CURRENT_SOURCE_DIR}/src/err-codes.h.in	${CMAKE_CURRENT_SOURCE_DIR}/src/errnos.in
			${CMAKE_CURRENT_SOURCE_DIR}/src/w32-add.h) # Windows specific

#[[
Keep this snippet here just in case we need it in the future
if(WIN32)
	configure_file(src/gpg-error.def.in _gpg-error.def.h)
	if(MSVC)
		add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/gpg-error.def
				COMMAND ${CMAKE_C_COMPILER} /DGEN_DEF /E /I ${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_BINARY_DIR}/_gpg-error.def.h |
				${GREP_EXE} -v "^#" > ${CMAKE_CURRENT_BINARY_DIR}/gpg-error.def)
	else()
		add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/gpg-error.def
				COMMAND ${CMAKE_C_COMPILER} -DGEN_DEF -E -I ${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_BINARY_DIR}/_gpg-error.def.h |
				${GREP_EXE} -v "^#" > ${CMAKE_CURRENT_BINARY_DIR}/gpg-error.def)
	endif()
	add_custom_target(gen_def DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/gpg-error.def)
endif()
]]

add_custom_target(gen_headers_gpgrt DEPENDS
${CMAKE_CURRENT_BINARY_DIR}/code-to-errno.h
${CMAKE_CURRENT_BINARY_DIR}/err-sources.h ${CMAKE_CURRENT_BINARY_DIR}/err-codes.h
${CMAKE_CURRENT_BINARY_DIR}/err-sources-sym.h ${CMAKE_CURRENT_BINARY_DIR}/err-codes-sym.h
${CMAKE_CURRENT_BINARY_DIR}/errnos-sym.h ${CMAKE_CURRENT_BINARY_DIR}/code-from-errno.h
${CMAKE_CURRENT_BINARY_DIR}/lock-obj-pub.native.h
${CMAKE_BINARY_DIR}/gpg-error.h ${CMAKE_BINARY_DIR}/gpgrt.h)

set(gpgrt_SOURCES
gettext.h protos.h gpgrt-int.h init.c init.h version.c lock.h thread.h
estream.c estream-printf.c estream-printf.h
strsource.c strerror.c code-to-errno.c code-from-errno.c
visibility.c visibility.h
sysutils.c stringutils.c syscall-clamp.c
logging.c b64dec.c b64enc.c argparse.c)

if(WIN32)
	list(APPEND gpgrt_SOURCES
	w32-gettext.c w32-lock.c w32-lock-obj.h w32-thread.c
	w32-iconv.c w32-estream.c w32-reg.c spawn-w32.c)
else()
	list(APPEND gpgrt_SOURCES
	posix-lock.c posix-lock-obj.h posix-thread.c spawn-posix.c)
endif()

if(WIN32)
	set(links ws2_32)
	list(APPEND gpgrt_SOURCES gpg-error.def)
else()
	set(links pthread)
endif()

list(TRANSFORM gpgrt_SOURCES PREPEND "src/")

add_library(gpgrt SHARED ${gpgrt_SOURCES})
add_dependencies(gpgrt gen_headers_gpgrt)
#if(WIN32)
#	add_dependencies(gpgrt gen_def)
#endif()
target_link_libraries(gpgrt ${links})
set_target_properties(gpgrt PROPERTIES PUBLIC_HEADER "${CMAKE_BINARY_DIR}/gpg-error.h;${CMAKE_BINARY_DIR}/gpgrt.h")
set_target_properties(gpgrt PROPERTIES POSITION_INDEPENDENT_CODE TRUE)
set_dll_output_directory(gpgrt ${CMAKE_BINARY_DIR})
add_library(GPGRT::GPGRT ALIAS gpgrt)

set(gpg_error_SOURCES strsource-sym.c strerror-sym.c gpg-error.c)
list(TRANSFORM gpg_error_SOURCES PREPEND "src/")
add_executable(gpg-error ${gpg_error_SOURCES})
target_link_libraries(gpg-error gpgrt)


install(TARGETS gpgrt gpg-error
		RUNTIME DESTINATION bin
		LIBRARY DESTINATION lib
		ARCHIVE DESTINATION lib
		PUBLIC_HEADER DESTINATION include)

file(GLOB gmo_files "po/*.gmo")
foreach(gmo ${gmo_files})
	string(REPLACE ${CMAKE_CURRENT_SOURCE_DIR}/po "" gmo ${gmo})
	string(REPLACE ".gmo" "" gmo ${gmo})
	install(FILES po/${gmo}.gmo DESTINATION share/locale/${gmo}/LC_MESSAGES/)
	install(CODE "file(RENAME ${CMAKE_INSTALL_PREFIX}/share/locale/${gmo}/LC_MESSAGES/${gmo}.gmo ${CMAKE_INSTALL_PREFIX}/share/locale/${gmo}/LC_MESSAGES/${PROJECT_NAME}.mo)")
endforeach()

#install(FILES doc/gpgrt.info DESTINATION share/info)
install(FILES doc/errorref.txt DESTINATION share/libgpg-error)

include(CTest)
if(BUILD_TESTING)

set(tests
t-version t-strerror t-syserror t-lock t-printf t-poll t-b64
t-argparse t-logging t-stringutils t-malloc)

foreach(t ${tests})
	add_executable(${t} tests/${t}.c)
	target_link_libraries(${t} gpgrt)
	add_test(NAME ${t} COMMAND ${t})
	if(WIN32)
		set_tests_properties(${t} PROPERTIES ENVIRONMENT "PATH=${CMAKE_BINARY_DIR}\;${NEW_PATH}")
	endif()
endforeach()

endif()
