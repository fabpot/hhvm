include(HHVMExtensionConfig)

set(RECURSIVE_SOURCE_SUBDIRS base debugger eval server vm)
foreach (dir ${RECURSIVE_SOURCE_SUBDIRS})
  HHVM_SELECT_SOURCES("${CMAKE_CURRENT_SOURCE_DIR}/${dir}")
endforeach(dir ${RECURSIVE_SOURCE_SUBDIRS})
list(APPEND HEADER_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/version.h")

if(NOT LINUX)
  list(REMOVE_ITEM CXX_SOURCES
       ${CMAKE_CURRENT_SOURCE_DIR}/vm/debug/elfwriter.cpp)
  list(REMOVE_ITEM HEADER_SOURCES
       ${CMAKE_CURRENT_SOURCE_DIR}/vm/debug/elfwriter.h)
endif()

if(NOT ENABLE_FASTCGI)
  HHVM_REMOVE_MATCHES_FROM_LISTS(CXX_SOURCES HEADER_SOURCES
    MATCHES "/fastcgi/")
endif()

if(NOT ENABLE_PROXYGEN_SERVER)
  HHVM_REMOVE_MATCHES_FROM_LISTS(CXX_SOURCES HEADER_SOURCES
    MATCHES "/proxygen/")
endif()

if(NOT HAVE_CUSTOM_LIBEVENT)
  HHVM_REMOVE_MATCHES_FROM_LISTS(CXX_SOURCES HEADER_SOURCES
    MATCHES "/server/libevent-")
endif()

add_custom_command(
  OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/ir-opcode-generated.h"
  COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/../doc/generate-ir-opcodes.sh"
  DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/../doc/ir.specification"
          "${CMAKE_CURRENT_SOURCE_DIR}/../doc/generate-ir-opcodes.pl"
          "${CMAKE_CURRENT_SOURCE_DIR}/../doc/generate-ir-opcodes.sh"
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  COMMENT "Generating HHIR opcode table"
  VERBATIM)

add_precompiled_header(
  "${HPHP_HOME}/hphp/pch/hphp_runtime_static.h"
  "${HPHP_HOME}/hphp/pch/hphp_runtime_static.cpp"
  CXX_SOURCES
)

add_object_library(hphp_runtime_static
            ${CXX_SOURCES} ${C_SOURCES} ${ASM_SOURCES} ${HEADER_SOURCES}
            "ir-opcode-generated.h")
auto_source_group("hphp_runtime_static" "${CMAKE_CURRENT_SOURCE_DIR}"
  ${ASM_SOURCES} ${C_SOURCES} ${CXX_SOURCES} ${HEADER_SOURCES})
SET_TARGET_PROPERTIES(hphp_runtime_static PROPERTIES OUTPUT_NAME "hphp_runtime")
SET_TARGET_PROPERTIES(hphp_runtime_static PROPERTIES PREFIX "lib")
SET_TARGET_PROPERTIES(hphp_runtime_static PROPERTIES CLEAN_DIRECT_OUTPUT 1)
object_library_hphp_link(hphp_runtime_static)
object_library_ld_link_libraries(hphp_runtime_static hphp_util hphp_proxygen vixl ppc64-asm hphp_zend hphp_system jit_sort)
HHVM_CONFIGURE_TARGET_FOR_EXTENSION_DEPENDENCIES(hphp_runtime_static)

SET(CMAKE_CXX_ARCHIVE_APPEND "<CMAKE_AR> q <TARGET> <LINK_FLAGS> <OBJECTS>")

HHVM_PUBLIC_HEADERS(runtime ${HEADER_SOURCES})

if (ENABLE_COTIRE)
  cotire(hphp_runtime_static)
endif()

set(EZC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../runtime/ext_zend_compat/")
include_directories("${EZC_DIR}/php-src")
include_directories("${EZC_DIR}/php-src/main")
include_directories("${EZC_DIR}/php-src/Zend")
include_directories("${EZC_DIR}/php-src/TSRM")
