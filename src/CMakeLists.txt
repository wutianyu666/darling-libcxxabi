# Get sources
set(LIBCXXABI_SOURCES
  abort_message.cpp
  cxa_aux_runtime.cpp
  cxa_default_handlers.cpp
  cxa_demangle.cpp
  cxa_exception.cpp
  cxa_exception_storage.cpp
  cxa_guard.cpp
  cxa_handlers.cpp
  cxa_new_delete.cpp
  cxa_personality.cpp
  cxa_unexpected.cpp
  cxa_vector.cpp
  cxa_virtual.cpp
  exception.cpp
  private_typeinfo.cpp
  stdexcept.cpp
  typeinfo.cpp
)

#if (UNIX AND NOT (APPLE OR CYGWIN))
#  list(APPEND LIBCXXABI_SOURCES cxa_thread_atexit.cpp)
#endif()

set(LIBCXXABI_HEADERS ../include/cxxabi.h)

# Add all the headers to the project for IDEs.
if (MSVC_IDE OR XCODE)
  # Force them all into the headers dir on MSVC, otherwise they end up at
  # project scope because they don't have extensions.
  if (MSVC_IDE)
    source_group("Header Files" FILES ${LIBCXXABI_HEADERS})
  endif()
endif()

include_directories("${LIBCXXABI_LIBCXX_INCLUDES}")

if (LIBCXXABI_HAS_CXA_THREAD_ATEXIT_IMPL)
  add_definitions(-DHAVE___CXA_THREAD_ATEXIT_IMPL)
endif()

# Generate library list.
set(libraries ${LIBCXXABI_CXX_ABI_LIBRARIES})
#append_if(libraries LIBCXXABI_HAS_C_LIB c)
#if (LIBCXXABI_ENABLE_THREADS)
#  append_if(libraries LIBCXXABI_HAS_PTHREAD_LIB pthread)
#endif()

#if (LIBCXXABI_USE_LLVM_UNWINDER)
#  list(APPEND libraries unwind)
#else()
#  append_if(libraries LIBCXXABI_HAS_GCC_EH_LIB gcc_eh)
#endif()

# Setup flags.
append_if(LIBCXXABI_COMPILE_FLAGS LIBCXXABI_HAS_FPIC_FLAG -fPIC )
append_if(LIBCXXABI_LINK_FLAGS LIBCXXABI_HAS_NODEFAULTLIBS_FLAG -nodefaultlibs)

if (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "i686")
	set(LIBCXXABI_SHARED_LINK_FLAGS "${LIBCXXABI_SHARED_LINK_FLAGS} -Wl,${CMAKE_CURRENT_SOURCE_DIR}/../i386-aliases")
endif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "i686")

if ( APPLE )
  if ( CMAKE_OSX_DEPLOYMENT_TARGET STREQUAL "10.6" )
    list(APPEND LIBCXXABI_COMPILE_FLAGS "-U__STRICT_ANSI__")
    list(APPEND LIBCXXABI_SHARED_LINK_FLAGS
      "-compatibility_version 1"
      "-current_version 1"
      "-install_name /usr/lib/libc++abi.1.dylib")
    list(APPEND LIBCXXABI_LINK_FLAGS
        "/usr/lib/libSystem.B.dylib")
  else()
    list(APPEND LIBCXXABI_SHARED_LINK_FLAGS
      "-compatibility_version 1"
      "-install_name /usr/lib/libc++abi.1.dylib")
  endif()
endif()

string(REPLACE ";" " " LIBCXXABI_COMPILE_FLAGS "${LIBCXXABI_COMPILE_FLAGS}")
string(REPLACE ";" " " LIBCXXABI_LINK_FLAGS "${LIBCXXABI_LINK_FLAGS}")
string(REPLACE ";" " " LIBCXXABI_SHARED_LINK_FLAGS "${LIBCXXABI_SHARED_LINK_FLAGS}")

# Add a object library that contains the compiled source files.
add_darling_object_library(cxxabi_objects ${LIBCXXABI_SOURCES} ${LIBCXXABI_HEADERS})

set_target_properties(cxxabi_objects
  PROPERTIES
    COMPILE_FLAGS "${LIBCXXABI_COMPILE_FLAGS} -Wno-gnu-include-next -Wno-undef -Wno-conversion -Wno-unused-parameter -Wno-sign-conversion"
  )
make_fat(cxxabi_objects)

set(LIBCXXABI_TARGETS)

SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-flat_namespace -Wl,-undefined,suppress")

# Build the shared library.
if (LIBCXXABI_ENABLE_SHARED)
  set(DYLIB_INSTALL_NAME "/usr/lib/libc++abi.dylib")
  add_darling_library(cxxabi_shared SHARED $<TARGET_OBJECTS:cxxabi_objects>)
  target_link_libraries(cxxabi_shared PRIVATE ${libraries} system_c system_kernel system_m unwind)
  make_fat(cxxabi_shared)
  set_target_properties(cxxabi_shared
    PROPERTIES
	#LINK_FLAGS    "${LIBCXXABI_LINK_FLAGS} ${LIBCXXABI_SHARED_LINK_FLAGS}"
      OUTPUT_NAME   "c++abi"
      VERSION       "1.0"
      SOVERSION     "1"
    )
  list(APPEND LIBCXXABI_TARGETS "cxxabi_shared")
endif()

# Build the static library.
if (LIBCXXABI_ENABLE_STATIC)
  set(CMAKE_AR "${CMAKE_BINARY_DIR}/src/external/cctools-port/cctools/ar/x86_64-apple-darwin11-ar")
  set(CMAKE_RANLIB "${CMAKE_BINARY_DIR}/src/external/cctools-port/cctools/misc/ranlib")
  add_library(cxxabi_static STATIC $<TARGET_OBJECTS:cxxabi_objects>)
  make_fat(cxxabi_static)
  target_link_libraries(cxxabi_static ${libraries})
  set_target_properties(cxxabi_static
    PROPERTIES
      LINK_FLAGS    "${LIBCXXABI_LINK_FLAGS}"
      OUTPUT_NAME   "c++abi"
  )
  list(APPEND LIBCXXABI_TARGETS "cxxabi_static")
  add_dependencies(cxxabi_static ranlib x86_64-apple-darwin11-ar)
endif()

# Add a meta-target for both libraries.
add_custom_target(cxxabi DEPENDS ${LIBCXXABI_TARGETS})

#install(TARGETS ${LIBCXXABI_TARGETS}
#  LIBRARY DESTINATION lib${LIBCXXABI_LIBDIR_SUFFIX} COMPONENT libcxxabi
#  ARCHIVE DESTINATION lib${LIBCXXABI_LIBDIR_SUFFIX} COMPONENT libcxxabi
#  )

install(TARGETS cxxabi_shared DESTINATION libexec/darling/usr/lib)

if (NOT CMAKE_CONFIGURATION_TYPES)
  add_custom_target(install-libcxxabi
    DEPENDS cxxabi
    COMMAND "${CMAKE_COMMAND}"
            -DCMAKE_INSTALL_COMPONENT=libcxxabi
            -P "${LIBCXXABI_BINARY_DIR}/cmake_install.cmake")
endif()
