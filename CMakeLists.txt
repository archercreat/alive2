cmake_minimum_required(VERSION 3.8)
include(ProcessorCount)

project(Alive2)

if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Debug)
endif()

message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

set(CMAKE_INSTALL_RPATH_USE_LINK_PATH ON)
set(CMAKE_BUILD_WITH_INSTALL_RPATH ON)

if (MSVC)
  set(CMAKE_CXX_FLAGS                "/GL /EHsc /W2 ${CMAKE_CXX_FLAGS}")
  set(CMAKE_CXX_FLAGS_DEBUG          "/Od /Zi ${CMAKE_CXX_FLAGS_DEBUG}")
  set(CMAKE_CXX_FLAGS_RELEASE        "/O2 /Oi /Oy /Zc:inline ${CMAKE_CXX_FLAGS_RELEASE}")
  set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "/O2 /Oi /Zi ${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
  set(CMAKE_EXE_LINKER_FLAGS         "/LTCG:INCREMENTAL ${CMAKE_EXE_LINKER_FLAGS}")
else()
  set(CMAKE_CXX_FLAGS                "-Wall -Werror -fPIC -fstrict-enums ${CMAKE_CXX_FLAGS}")
  set(CMAKE_CXX_FLAGS_DEBUG          "-g -Og ${CMAKE_CXX_FLAGS_DEBUG}")
  set(CMAKE_CXX_FLAGS_RELEASE        "-O3 ${CMAKE_CXX_FLAGS_RELEASE}")
  set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELEASE} -g ${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
endif()

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

macro(remove_flags)
  foreach(var ${ARGN})
    string(REPLACE "${var}" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    string(REPLACE "${var}" "" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
    string(REPLACE "${var}" "" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
    string(REPLACE "${var}" "" CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
    remove_definitions(${var})
  endforeach()
endmacro(remove_flags)

remove_flags(-DNDEBUG)

list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")

if (CYGWIN)
  set(CMAKE_FIND_LIBRARY_SUFFIXES ".dll")
endif()

find_package(Git REQUIRED)
add_custom_command(
  OUTPUT "${PROJECT_BINARY_DIR}/version_gen.h.tmp"
  COMMAND "${CMAKE_COMMAND}" -E echo_append "#define ALIVE_VERSION_MACRO " > "${PROJECT_BINARY_DIR}/version_gen.h.tmp"
  COMMAND "${GIT_EXECUTABLE}" describe --tags --dirty --always >> "${PROJECT_BINARY_DIR}/version_gen.h.tmp"
  WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
  VERBATIM
)
add_custom_command(
  DEPENDS "${PROJECT_BINARY_DIR}/version_gen.h.tmp"
  OUTPUT "${PROJECT_BINARY_DIR}/version_gen.h"
  COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${PROJECT_BINARY_DIR}/version_gen.h.tmp" "${PROJECT_BINARY_DIR}/version_gen.h"
  COMMAND "${CMAKE_COMMAND}" -E remove -f "${PROJECT_BINARY_DIR}/version_gen.h.tmp"
  WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
  VERBATIM
)
add_custom_target(
  generate_version ALL
  DEPENDS "${PROJECT_BINARY_DIR}/version_gen.h"
  WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
)

find_package(ZLIB)
find_package(Z3 4.8.5 REQUIRED)
include_directories(${Z3_INCLUDE_DIR})

find_program(RE2C re2c)
if (RE2C)
  message(STATUS "RE2C: ${RE2C}")
else()
  message(SEND_ERROR "re2c executable not found")
endif()
add_custom_command(OUTPUT "${PROJECT_BINARY_DIR}/tools/alive_lexer.cpp"
                   COMMAND ${CMAKE_COMMAND} -E make_directory "${PROJECT_BINARY_DIR}/tools"
                   COMMAND ${RE2C} ARGS "-d" "-b" "-T" "--no-generation-date"
                   "-o" "${PROJECT_BINARY_DIR}/tools/alive_lexer.cpp"
                   "${PROJECT_SOURCE_DIR}/tools/alive_lexer.re"
                   DEPENDS "tools/alive_lexer.re"
)


file(GLOB_RECURSE IR_SOURCES  CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/ir/*.cpp")
file(GLOB_RECURSE IR_INCLUDES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/ir/*.h")

file(GLOB_RECURSE SMT_SOURCES  CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/smt/*.cpp")
file(GLOB_RECURSE SMT_INCLUDES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/smt/*.h")

file(GLOB_RECURSE TOOLS_INCLUDES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/tools/*.h")

file(GLOB_RECURSE UTIL_SOURCES  CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/util/*.cpp")
file(GLOB_RECURSE UTIL_INCLUDES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/util/*.h")

file(GLOB_RECURSE CACHE_SOURCES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/cache/*.cpp")

file(GLOB_RECURSE LLVM_UTIL_SOURCES  CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/llvm_util/*.cpp")
file(GLOB_RECURSE LLVM_UTIL_INCLUDES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/llvm_util/*.h")

add_library(ir STATIC ${IR_SOURCES})

target_include_directories(ir PUBLIC
  $<INSTALL_INTERFACE:include/alive>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
)

add_library(smt STATIC ${SMT_SOURCES})

target_include_directories(smt PUBLIC
  $<INSTALL_INTERFACE:include/alive>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
)

add_library(tools STATIC "tools/transform.cpp")

target_include_directories(tools PUBLIC
  $<INSTALL_INTERFACE:include/alive>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
)

add_library(util STATIC ${UTIL_SOURCES})
target_include_directories(util PUBLIC
  $<INSTALL_INTERFACE:include/alive>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
)
target_include_directories(util PRIVATE $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}>)
add_dependencies(util generate_version)

set(ALIVE_LIBS ir smt tools util)

find_package(hiredis)
if (HIREDIS_LIBRARIES)
  add_library(cache STATIC ${CACHE_SOURCES})
  target_include_directories(cache PRIVATE ${HIREDIS_INCLUDE_DIR})
  target_include_directories(cache PUBLIC ${PROJECT_SOURCE_DIR})
  set(ALIVE_LIBS cache ${ALIVE_LIBS})
else()
  set(HIREDIS_LIBRARIES $<0:''>)
  add_compile_definitions(NO_REDIS_SUPPORT)
endif()


if (BUILD_LLVM_UTILS OR BUILD_TV)
  find_package(LLVM REQUIRED CONFIG)

  message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
  message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")

  if (NOT LLVM_ENABLE_RTTI)
    message(FATAL_ERROR "LLVM must be built with '-DLLVM_ENABLE_RTTI=ON'")
  endif()

  if (NOT LLVM_ENABLE_EH)
    message(FATAL_ERROR "LLVM must be built with '-DLLVM_ENABLE_EH=ON'")
  endif()

  list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")
  include(AddLLVM)

  add_library(llvm_util STATIC ${LLVM_UTIL_SOURCES})

  target_include_directories(llvm_util PRIVATE ${LLVM_INCLUDE_DIRS})
  target_include_directories(llvm_util PUBLIC
    $<INSTALL_INTERFACE:include/alive>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  )
  add_definitions(${LLVM_DEFINITIONS})
  set(ALIVE_LIBS_LLVM llvm_util ${ALIVE_LIBS})

  set(LLVM_LINK_COMPONENTS ${LLVM_LINK_COMPONENTS} TargetParser)

  add_llvm_executable(alive-tv
    "tools/alive-tv.cpp"
  )

  add_llvm_executable(quick-fuzz
    "tools/quick-fuzz.cpp"
  )

  add_llvm_executable(alive-exec
    "tools/alive-exec.cpp"
  )
endif()

if (BUILD_TV)
  if (CYGWIN)
    message(FATAL_ERROR "LLVM plugins not supported on cygwin")
  endif()
  message(STATUS "Compiling translation validation plugin")
  add_subdirectory(tv)
  set(FOR_ALIVE2_TEST 0)
  configure_file(
    ${CMAKE_SOURCE_DIR}/scripts/opt-alive.sh.in
    ${CMAKE_BINARY_DIR}/opt-alive.sh
    @ONLY
  )
  set(FOR_ALIVE2_TEST 1)
  configure_file(
    ${CMAKE_SOURCE_DIR}/scripts/opt-alive.sh.in
    ${CMAKE_BINARY_DIR}/opt-alive-test.sh
    @ONLY
  )
  unset(FOR_ALIVE2_TEST)
else()
  message(STATUS "Skipping translation validation plugin")
endif()

add_executable(alive
               "tools/alive.cpp"
               "${PROJECT_BINARY_DIR}/tools/alive_lexer.cpp"
               "tools/alive_parser.cpp"
              )
target_link_libraries(alive PRIVATE ${ALIVE_LIBS})

add_executable(alive-jobserver
               "tools/alive-jobserver.cpp"
              )

#add_library(alive2 SHARED ${IR_SRCS} ${SMT_SRCS} ${TOOLS_SRCS} ${UTIL_SRCS} ${LLVM_UTIL_SRCS})

if (BUILD_LLVM_UTILS OR BUILD_TV)
  llvm_map_components_to_libnames(llvm_libs support core irreader bitwriter analysis passes transformutils)
#  target_link_libraries(alive2 PRIVATE ${llvm_libs})
  target_link_libraries(alive-tv PRIVATE ${ALIVE_LIBS_LLVM} ${Z3_LIBRARIES} ${HIREDIS_LIBRARIES} ${llvm_libs})
  target_link_libraries(quick-fuzz PRIVATE ${ALIVE_LIBS_LLVM} ${Z3_LIBRARIES} ${HIREDIS_LIBRARIES} ${llvm_libs})
  target_link_libraries(alive-exec PRIVATE ${ALIVE_LIBS_LLVM} ${Z3_LIBRARIES} ${HIREDIS_LIBRARIES} ${llvm_libs})
endif()

target_link_libraries(alive PRIVATE ${Z3_LIBRARIES} ${HIREDIS_LIBRARIES})
#target_link_libraries(alive2 PRIVATE ${Z3_LIBRARIES} ${HIREDIS_LIBRARIES})

if (NOT DEFINED TEST_NTHREADS)
  ProcessorCount(TEST_NTHREADS)
  if (TEST_NTHREADS EQUAL 0)
    set(TEST_NTHREADS 1)
  endif()
endif()
add_custom_target("check"
                  COMMAND "python3"
                          "${PROJECT_SOURCE_DIR}/tests/lit/lit.py"
                          "-s"
                          "${PROJECT_SOURCE_DIR}/tests"
                          "-j${TEST_NTHREADS}"
                  DEPENDS "alive"
                  USES_TERMINAL
                 )

if (BUILD_TV)
  add_dependencies("check" "alive-tv" "quick-fuzz" "tv")
endif()

install(
  EXPORT aliveTargets
  NAMESPACE alive2::
  DESTINATION lib/cmake/alive2
)

install(
  TARGETS ir smt tools util alive alive-jobserver
  EXPORT aliveTargets
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
)

install(FILES ${IR_INCLUDES}    DESTINATION "include/alive2/ir")
install(FILES ${SMT_INCLUDES}   DESTINATION "include/alive2/smt")
install(FILES ${TOOLS_INCLUDES} DESTINATION "include/alive2/tools")
install(FILES ${UTIL_INCLUDES}  DESTINATION "include/alive2/util")

if(TARGET llvm_util)
  install(
    TARGETS llvm_util alive-tv alive-exec quick-fuzz
    EXPORT aliveTargets
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
  )
  install(FILES ${LLVM_UTIL_INCLUDES} DESTINATION "include/alive2/llvm_util")
endif()

install(
  FILES
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/aliveConfig.cmake"
  DESTINATION lib/cmake/alive2
)
