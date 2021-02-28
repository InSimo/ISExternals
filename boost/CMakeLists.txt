cmake_minimum_required(VERSION 2.8.12)

project(BoostWin)
option(BOOST_INSTALL_HEADERS "" OFF )
set(BOOST_ROOT ${CMAKE_CURRENT_SOURCE_DIR} CACHE PATH "Directory for Boost find_package")

set(Boost_NO_SYSTEM_PATHS ON CACHE BOOL "")
set(Boost_USE_STATIC_LIBS OFF CACHE BOOL "")
set(Boost_USE_MULTITHREADED ON CACHE BOOL "")
set(Boost_USE_STATIC_RUNTIME OFF CACHE BOOL "")

include(GNUInstallDirs)
set(BOOST_INSTALL_BIN_DIR     ${CMAKE_INSTALL_BINDIR}     CACHE PATH "Installation directory for binaries relative to CMAKE_INSTALL_PREFIX variable")
set(BOOST_INSTALL_LIB_DIR     ${CMAKE_INSTALL_LIBDIR}     CACHE PATH "Installation directory for libraries relative to CMAKE_INSTALL_PREFIX variable")
set(BOOST_INSTALL_INCLUDE_DIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE PATH "Installation directory for header files relative to CMAKE_INSTALL_PREFIX variable")

if (WIN32)
  set( BOOST_INCLUDEDIR "${BOOST_ROOT}" CACHE PATH "")
  if(CMAKE_CL_64)
    set( BOOST_LIBARCH "lib64")
  else()
    set( BOOST_LIBARCH "lib32")
  endif()
  if (MSVC10)
    set( BOOST_LIBCOMP "-msvc-10.0")
  endif()
  if (MSVC11)
    set( BOOST_LIBCOMP "-msvc-11.0")
  endif()
  if (MSVC12)
    set( BOOST_LIBCOMP "-msvc-12.0")
  endif()
  if (MSVC14)
    set( BOOST_LIBCOMP "-msvc-14.0")
  endif()
  if (MSVC15)
    set( BOOST_LIBCOMP "-msvc-15.0")
  endif()
  set( BOOST_LIBRARYDIR "${BOOST_ROOT}/${BOOST_LIBARCH}${BOOST_LIBCOMP}" CACHE PATH "")
endif()

## Boost
#find_package(Boost COMPONENTS thread system date_time filesystem locale QUIET)
find_package(Boost REQUIRED)


file(GLOB BOOST_DLLS "${BOOST_LIBRARYDIR}/*-mt-1*.dll")
file(GLOB BOOST_LIBS "${BOOST_LIBRARYDIR}/*-mt-1*.lib")
file(GLOB BOOST_DLLS_DEBUG "${BOOST_LIBRARYDIR}/*-mt-gd-1*.dll")
file(GLOB BOOST_LIBS_DEBUG "${BOOST_LIBRARYDIR}/*-mt-gd-1*.lib")

set(CONFIGS ${CMAKE_CONFIGURATION_TYPES})

foreach(CONFIG ${CONFIGS})
    file(MAKE_DIRECTORY "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${CONFIG}")
    if ("${CONFIG}" STREQUAL "Debug")
        foreach(DLL ${BOOST_DLLS_DEBUG})
            file(COPY ${DLL} DESTINATION "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${CONFIG}")
        endforeach()
        install( FILES ${BOOST_DLLS_DEBUG} DESTINATION  ${BOOST_INSTALL_BIN_DIR} CONFIGURATIONS ${CONFIG} )
        install( FILES ${BOOST_LIBS_DEBUG} DESTINATION  ${BOOST_INSTALL_LIB_DIR} CONFIGURATIONS ${CONFIG} )
    else()
        foreach(DLL ${BOOST_DLLS})
            file(COPY ${DLL} DESTINATION "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${CONFIG}")
        endforeach()
        install( FILES ${BOOST_DLLS} DESTINATION  ${BOOST_INSTALL_BIN_DIR} CONFIGURATIONS ${CONFIG} )
        install( FILES ${BOOST_LIBS} DESTINATION  ${BOOST_INSTALL_LIB_DIR} CONFIGURATIONS ${CONFIG} )
    endif()
endforeach()

if( BOOST_INSTALL_HEADERS )
    install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/boost
            DESTINATION "${BOOST_INSTALL_INCLUDE_DIR}"
            FILES_MATCHING 
                PATTERN "*.h"
                PATTERN "*.inl"
                PATTERN "*.hxx"
                PATTERN "*.hpp"
                PATTERN "*.ipp"
                PATTERN ".svn" EXCLUDE
            )
endif()