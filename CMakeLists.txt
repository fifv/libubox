cmake_minimum_required(VERSION 3.13)
include(CheckLibraryExists)
include(CheckFunctionExists)

project(ubox C)

add_definitions(-Wall -Werror)
if(CMAKE_C_COMPILER_VERSION VERSION_GREATER 6)
	add_definitions(-Wextra -Werror=implicit-function-declaration)
	add_definitions(-Wformat -Werror=format-security -Werror=format-nonliteral)
endif()
add_definitions(-Os -std=gnu99 -g3 -Wmissing-declarations -Wno-unused-parameter)

option(BUILD_LUA "build Lua plugin" OFF)
option(BUILD_EXAMPLES "build examples" ON)

include(FindPkgConfig)
pkg_search_module(JSONC json-c REQUIRED)
include_directories(${JSONC_INCLUDE_DIRS})


add_library(ubox avl.c avl-cmp.c blob.c blobmsg.c uloop.c usock.c ustream.c ustream-fd.c vlist.c utils.c safe_list.c runqueue.c md5.c kvlist.c ulog.c base64.c udebug.c udebug-remote.c)
target_include_directories(ubox PUBLIC ${CMAKE_SOURCE_DIR})

set(LIBS)
check_function_exists(clock_gettime HAVE_GETTIME)
check_function_exists(shm_open HAVE_SHM)
if(NOT HAVE_GETTIME OR NOT HAVE_SHM)
	check_library_exists(rt clock_gettime "" NEED_GETTIME)
	check_library_exists(rt shm_open "" NEED_SHM)
	if(NEED_GETTIME OR NEED_SHM)
		target_link_libraries(ubox rt)
	endif()
endif()

file(GLOB headers *.h)
list(FILTER headers EXCLUDE REGEX "-priv.h$" )
install(FILES ${headers}
	DESTINATION include/libubox
)
install(TARGETS ubox
	ARCHIVE DESTINATION lib
	LIBRARY DESTINATION lib
)

# add_subdirectory(lua)
add_subdirectory(examples)

macro(ADD_UNIT_TEST_SAN name)
  add_executable(${name}-san ${name}.c)
  target_compile_options(${name}-san PRIVATE -g -fno-omit-frame-pointer -fsanitize=undefined,address,leak -fno-sanitize-recover=all)
  target_link_options(${name}-san PRIVATE -fsanitize=undefined,address,leak)
  target_link_libraries(${name}-san ubox blobmsg_json json_script ${json})
  target_include_directories(${name}-san PRIVATE ${PROJECT_SOURCE_DIR})
endmacro(ADD_UNIT_TEST_SAN)

if(UNIT_TESTING)
  enable_testing()
  add_subdirectory(tests)
endif()

find_library(json NAMES json-c)
if(EXISTS ${json})
	add_library(blobmsg_json SHARED blobmsg_json.c)
	target_link_libraries(blobmsg_json ubox ${json})

	add_library(blobmsg_json-static STATIC blobmsg_json.c)
	set_target_properties(blobmsg_json-static
			      PROPERTIES OUTPUT_NAME blobmsg_json)

	if(UNIT_TESTING)
		add_unit_test_san(jshn)
	endif(UNIT_TESTING)

	add_executable(jshn jshn.c)
	target_link_libraries(jshn blobmsg_json ${json})

	add_library(json_script SHARED json_script.c)
	target_link_libraries(json_script ubox)

	install(TARGETS blobmsg_json blobmsg_json-static jshn json_script
		ARCHIVE DESTINATION lib
		LIBRARY DESTINATION lib
		RUNTIME DESTINATION bin
	)

	file(GLOB scripts sh/*.sh)
	install(FILES ${scripts}
		DESTINATION share/libubox
	)

endif()

if(ABIVERSION)
	set_target_properties(ubox PROPERTIES VERSION ${ABIVERSION})
	set_target_properties(json_script PROPERTIES VERSION ${ABIVERSION})
	set_target_properties(blobmsg_json PROPERTIES VERSION ${ABIVERSION})
endif()

add_custom_target(debian
	COMMAND ${CMAKE_COMMAND} -E echo "Generating debian/changelog from git..."
	COMMAND ${CMAKE_SOURCE_DIR}/debian/generate-changelog.sh
	COMMAND dpkg-buildpackage -b -uc -us
	WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
	COMMENT "Building Debian package"
)
