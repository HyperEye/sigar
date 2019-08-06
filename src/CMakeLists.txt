## sigar has some base files + a set of platform specific files

MESSAGE(STATUS "CMAKE_SYSTEM_NAME is ${CMAKE_SYSTEM_NAME}")

INCLUDE_DIRECTORIES(../include/)

INCLUDE(CheckCSourceCompiles)

MACRO (CHECK_STRUCT_MEMBER _STRUCT _MEMBER _HEADER _RESULT)
   SET(_INCLUDE_FILES)
   FOREACH (it ${_HEADER})
      SET(_INCLUDE_FILES "${_INCLUDE_FILES}#include <${it}>\n")
   ENDFOREACH (it)

   SET(_CHECK_STRUCT_MEMBER_SOURCE_CODE "
${_INCLUDE_FILES}
int main()
{
   static ${_STRUCT} tmp;
   if (sizeof(tmp.${_MEMBER}))
      return 0;
  return 0;
}
")
   CHECK_C_SOURCE_COMPILES("${_CHECK_STRUCT_MEMBER_SOURCE_CODE}" ${_RESULT})

ENDMACRO (CHECK_STRUCT_MEMBER)

MACRO (CHECK_SYMBOL_EXISTS _SYMBOL _HEADER _LIB _RESULT)
   SET(_INCLUDE_FILES)
   FOREACH (it ${_HEADER})
      SET(_INCLUDE_FILES "${_INCLUDE_FILES}#include <${it}>\n")
   ENDFOREACH (it)

   SET(_CHECK_SYMBOL_EXISTS_SOURCE_CODE "
${_INCLUDE_FILES}
int main()
{
void* tmp = (void*)&${_SYMBOL};
   return 0;
}
")
   SET(CMAKE_REQUIRED_LIBRARIES ${_LIB})
   CHECK_C_SOURCE_COMPILES("${_CHECK_SYMBOL_EXISTS_SOURCE_CODE}" ${_RESULT})

ENDMACRO (CHECK_SYMBOL_EXISTS)

## linux
IF(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  SET(SIGAR_SRC os/linux/linux_sigar.c)
  SET(SIGAR_HDR os/linux/sigar_os.h)

  INCLUDE_DIRECTORIES(os/linux/)
ENDIF(CMAKE_SYSTEM_NAME STREQUAL "Linux")

## macosx, freebsd
IF(CMAKE_SYSTEM_NAME MATCHES "(Darwin|FreeBSD)")
  SET(SIGAR_SRC os/darwin/darwin_sigar.c)
  SET(SIGAR_HDR
	os/darwin/if_types.h
	os/darwin/sigar_os.h
	os/darwin/tcp_fsm.h
  )

  INCLUDE_DIRECTORIES(os/darwin/)
  IF(CMAKE_SYSTEM_NAME MATCHES "(Darwin)")
    ADD_DEFINITIONS(-DDARWIN)
    SET(SIGAR_LINK_FLAGS "-framework CoreServices -framework IOKit")
  ELSE(CMAKE_SYSTEM_NAME MATCHES "(Darwin)")
    ## freebsd needs libkvm
    SET(SIGAR_LINK_FLAGS "-lkvm")
  ENDIF(CMAKE_SYSTEM_NAME MATCHES "(Darwin)")
ENDIF(CMAKE_SYSTEM_NAME MATCHES "(Darwin|FreeBSD)")

## solaris
IF (CMAKE_SYSTEM_NAME MATCHES "(Solaris|SunOS)" )
  SET(SIGAR_SRC
	os/solaris/solaris_sigar.c
	os/solaris/get_mib2.c
	os/solaris/kstats.c
	os/solaris/procfs.c
  )
SET(SIGAR_HDR
	os/solaris/get_mib2.h
	os/solaris/sigar_os.h
  )

  INCLUDE_DIRECTORIES(os/solaris/)
  ADD_DEFINITIONS(-DSOLARIS)
  ADD_DEFINITIONS(-DHAVE_LIBDLPI_H)
  SET(SIGAR_LINK_FLAGS "-lkstat -ldl -lnsl -lsocket -lresolv -ldlpi")
ENDIF(CMAKE_SYSTEM_NAME MATCHES "(Solaris|SunOS)" )

## hpux
IF (CMAKE_SYSTEM_NAME MATCHES "(hpux)" )
  SET(SIGAR_SRC os/hpux/hpux_sigar.c)
  SET(SIGAR_HDR os/hpux/sigar_os.h)
  INCLUDE_DIRECTORIES(os/hpux/)
  ADD_DEFINITIONS(-DSIGAR_HPUX)
  SET(SIGAR_LINK_FLAGS -lnm)
ENDIF(CMAKE_SYSTEM_NAME MATCHES "(hpux)" )

## aix
IF (CMAKE_SYSTEM_NAME MATCHES "(AIX)" )
  SET(SIGAR_SRC os/aix/aix_sigar.c)
  SET(SIGAR_HDR os/aix/sigar_os.h)

  INCLUDE_DIRECTORIES(os/aix/)
  SET(SIGAR_LINK_FLAGS -lodm -lcfg)
ENDIF(CMAKE_SYSTEM_NAME MATCHES "(AIX)" )

IF(WIN32)
  ADD_DEFINITIONS(-DSIGAR_SHARED)
  SET(SIGAR_SRC os/win32/peb.c os/win32/win32_sigar.c os/win32/wmi.cpp)
  SET(SIGAR_HDR
	os/win32/sigar_os.h
	os/win32/sigar_pdh.h
  )
  INCLUDE_DIRECTORIES(os/win32)
  CHECK_STRUCT_MEMBER(MIB_IPADDRROW wType "windows.h;iphlpapi.h" wType_in_MIB_IPADDRROW)
  if(${wType_in_MIB_IPADDRROW})
      add_definitions(-DHAVE_MIB_IPADDRROW_WTYPE)
  endif()
  CHECK_SYMBOL_EXISTS(CLSID_WbemLocator "windows.h;ole2.h;wbemcli.h" "wbemuuid"  HAS_WBEMLOCATOR)
  if(${HAS_WBEMLOCATOR})
    add_definitions(-DHAVE_WBEMLOCATOR)
  endif()
ENDIF(WIN32)

SET(SIGAR_SRC ${SIGAR_SRC}
  sigar.c
  sigar_cache.c
  sigar_elf.c
  sigar_fileinfo.c
  sigar_format.c
  sigar_ptql.c
  sigar_rma.c
  sigar_signal.c
  sigar_util.c
)

SET(SIGAR_HDR ${SIGAR_HDR}
  ../include/sigar_fileinfo.h
  ../include/sigar_format.h
  ../include/sigar.h
  ../include/sigar_log.h
  ../include/sigar_private.h
  ../include/sigar_ptql.h
  ../include/sigar_util.h
  ../include/machine/endian.h
)

ADD_LIBRARY(sigar SHARED ${SIGAR_SRC} ${SIGAR_HDR})
IF(WIN32)
	TARGET_LINK_LIBRARIES(sigar ws2_32 netapi32 version)
ENDIF(WIN32)
IF(SIGAR_LINK_FLAGS)
  SET_TARGET_PROPERTIES(sigar PROPERTIES LINK_FLAGS "${SIGAR_LINK_FLAGS}")
ENDIF(SIGAR_LINK_FLAGS)

INSTALL(TARGETS sigar
	RUNTIME DESTINATION bin
	ARCHIVE DESTINATION lib
	LIBRARY DESTINATION lib
	)
