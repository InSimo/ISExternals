cmake_minimum_required(VERSION 3.0)

# This file should be included early in the main CMakeLists.txt so that the
# Python we embed is properly set-up before doing any python-related stuff.

if( WIN32 )
  # On Windows we embed a distribution of Python to be self-contained
  # and not having to install it
  set( EMBEDDED_PYTHON TRUE CACHE INTERNAL "" FORCE )
  if( CMAKE_CL_64 )
    set( _ARCH_SUBDIR "win-x64" )
  else()
    set( _ARCH_SUBDIR "win-x86" )
  endif()

  # First, we make a working copy of the whole Python distribution into the build tree
  # (to avoid touching the sources, like having pyc files generated)
  set( SOURCE_PYTHON_DIR "${CMAKE_CURRENT_SOURCE_DIR}/${_ARCH_SUBDIR}" )
  set( TARGET_PYTHON_DIR "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}" )

  if( NOT EXISTS "${TARGET_PYTHON_DIR}/python.exe" )
      message( STATUS "Copying the embedded python from ${SOURCE_PYTHON_DIR} to ${TARGET_PYTHON_DIR}" )
      file( GLOB SOURCE_PYTHON_FILES "${SOURCE_PYTHON_DIR}/*" )
      file( COPY ${SOURCE_PYTHON_FILES} DESTINATION ${TARGET_PYTHON_DIR}
            PATTERN "*.pyc" EXCLUDE
            PATTERN "Scripts" EXCLUDE
            PATTERN "README.txt" EXCLUDE
            PATTERN "NEWS.txt" EXCLUDE
            PATTERN "LICENSE.txt" EXCLUDE
            )
  else()
    message( STATUS "Found python executable in the build tree: ${TARGET_PYTHON_DIR}/python.exe - skip copy")
  endif()

  # copy and rename LICENSE.txt
  configure_file("${SOURCE_PYTHON_DIR}/LICENSE.txt" "${TARGET_PYTHON_DIR}/Python-LICENSE.txt" COPYONLY)

  set( EMBEDDED_PYTHON_DIR ${TARGET_PYTHON_DIR} CACHE INTERNAL "" FORCE )

  # Set the PYTHON_EXECUTABLE var to force using our embedded Python for the configuration
  # and the build instead of the one possibly installed on the system.
  # (setting the var as CACHE INTERNAL, otherwise find_package(PythonInterp) can fail,
  # and it is also set this way in the official FindPython*.cmake)
  set( PYTHON_EXECUTABLE "${EMBEDDED_PYTHON_DIR}/python.exe" CACHE INTERNAL "" FORCE )
  if( NOT EXISTS ${PYTHON_EXECUTABLE} )
    message( FATAL_ERROR "The embedded python executable was not found at ${PYTHON_EXECUTABLE}" )
    return()
  endif()

  set( PYTHONW_EXECUTABLE "${EMBEDDED_PYTHON_DIR}/pythonw.exe" )

  file( GLOB EMBEDDED_PYTHON_DLL "${EMBEDDED_PYTHON_DIR}/*.dll" )
  set( EMBEDDED_PYTHON_DLL ${EMBEDDED_PYTHON_DLL} CACHE INTERNAL "" FORCE )

  # Copy python.exe and the python DLL in each build config directory, to have them
  # in the same folder as the other executables and DLLs. This way we can avoid some
  # headache with DLLs not always found at runtime.
  # bin/<config>/python.exe will correctly use 'Lib' and 'DLLs' folders in the parent
  # dir, so we don't have to copy other files.
  foreach( CONFIG ${CMAKE_CONFIGURATION_TYPES} )
    file( COPY ${PYTHON_EXECUTABLE} ${EMBEDDED_PYTHON_DLL} ${PYTHONW_EXECUTABLE}
          DESTINATION "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${CONFIG}" )
  endforeach()

  # We will also install python in the install bin dir
  include(GNUInstallDirs)
  set( EMBEDDED_PYTHON_INSTALL_DIR ${CMAKE_INSTALL_BINDIR} CACHE INTERNAL "" FORCE )
  install( DIRECTORY "${SOURCE_PYTHON_DIR}/" # trailing slash to avoid having an extra folder level in the copy
           DESTINATION "${EMBEDDED_PYTHON_INSTALL_DIR}"
           PATTERN "*.pyc" EXCLUDE
           PATTERN "include" EXCLUDE
           PATTERN "libs" EXCLUDE
           PATTERN "Scripts" EXCLUDE
           PATTERN "README.txt" EXCLUDE
           PATTERN "NEWS.txt" EXCLUDE
           PATTERN "LICENSE.txt" EXCLUDE )
  install( FILES "${TARGET_PYTHON_DIR}/Python-LICENSE.txt"
           DESTINATION ${EMBEDDED_PYTHON_INSTALL_DIR} )

  if( ${CMAKE_VERSION} VERSION_LESS "3.12" )
    find_package( PythonInterp 2.7 REQUIRED )
    find_package( PythonLibs 2.7 REQUIRED )
  else()
    # CMake >= 3.12: find_package(PythonInterp ...) is deprecated
    set( Python_ROOT_DIR ${EMBEDDED_PYTHON_DIR} CACHE INTERNAL "" FORCE )
    set( Python2_ROOT_DIR ${EMBEDDED_PYTHON_DIR} CACHE INTERNAL "" FORCE )
    find_package( Python2 COMPONENTS Interpreter Development REQUIRED )
  endif()

else()
  # We would like the same version as on Windows, but let's wait for the Anaconda open-source distribution
  # to be available for this version
  set( REQUIRED_PYTHON_VERSION 2.7 )
  set( EMBEDDED_PYTHON TRUE CACHE INTERNAL "" FORCE )
  if( CMAKE_SIZEOF_VOID_P EQUAL 8 )
    set( _ARCH_SUBDIR "linux-x64" )
  else()
    set( _ARCH_SUBDIR "linux-x86" )
  endif()

  # First, we make a working copy of the whole Python distribution into the build tree
  # (to avoid touching the sources, like having pyc files generated)
  set( SOURCE_PYTHON_DIR "${CMAKE_CURRENT_SOURCE_DIR}/${_ARCH_SUBDIR}" )
  set( TARGET_PYTHON_DIR "${CMAKE_BINARY_DIR}" )

  if( EXISTS ${TARGET_PYTHON_DIR}/bin/activate )
    # If there is an existing virtual env in the build tree (formerly we used to install such environment),
    # it can cause issues when trying to detect the new full Python distribution
    message( STATUS "A previous virtual env installation was detected in ${TARGET_PYTHON_DIR}, wiping it..." )
    file( REMOVE "${TARGET_PYTHON_DIR}/pyvenv.cfg"
                 "${TARGET_PYTHON_DIR}/bin/python*"
                 "${TARGET_PYTHON_DIR}/bin/activate*"
                 "${TARGET_PYTHON_DIR}/bin/pip*"
                 "${TARGET_PYTHON_DIR}/bin/easy_install*"
                 "${TARGET_PYTHON_DIR}/lib/python*.*"
                 "${TARGET_PYTHON_DIR}/lib/libpython*" )
  endif()

  message( STATUS "Copying the embedded python from ${SOURCE_PYTHON_DIR} to ${TARGET_PYTHON_DIR}" )

  file( GLOB SOURCE_PYTHON_FILES "${SOURCE_PYTHON_DIR}/*" )
  file( COPY ${SOURCE_PYTHON_FILES} DESTINATION ${TARGET_PYTHON_DIR}
        PATTERN "*.pyc" EXCLUDE
        PATTERN "__pycache__" EXCLUDE
        PATTERN "bin/python*" EXCLUDE )

  # Copy the python executable with exec rights
  file(COPY "${SOURCE_PYTHON_DIR}/bin/python" DESTINATION "${TARGET_PYTHON_DIR}/bin"
       FILE_PERMISSIONS OWNER_EXECUTE OWNER_WRITE OWNER_READ)
  file(COPY "${SOURCE_PYTHON_DIR}/bin/python2" DESTINATION "${TARGET_PYTHON_DIR}/bin"
       FILE_PERMISSIONS OWNER_EXECUTE OWNER_WRITE OWNER_READ)

  # copy and rename LICENSE.txt
  configure_file("${SOURCE_PYTHON_DIR}/LICENSE.txt" "${TARGET_PYTHON_DIR}/Anaconda-LICENSE.txt" COPYONLY)

  set( EMBEDDED_PYTHON_DIR ${TARGET_PYTHON_DIR} CACHE INTERNAL "" FORCE )

  # Set the PYTHON_EXECUTABLE var to force using our embedded Python for the configuration
  # and the build instead of the one possibly installed on the system.
  # (setting the var as CACHE INTERNAL, otherwise find_package(PythonInterp) can fail,
  # and it is also set this way in the official FindPython*.cmake)
  set( PYTHON_EXECUTABLE "${EMBEDDED_PYTHON_DIR}/bin/python" CACHE INTERNAL "" FORCE )
  if( NOT EXISTS ${PYTHON_EXECUTABLE} )
    message( FATAL_ERROR "The embedded python executable was not found at ${PYTHON_EXECUTABLE}" )
    return()
  else()
    message(STATUS "Set PYTHON_EXECUTABLE=${PYTHON_EXECUTABLE}")
  endif()

  execute_process(COMMAND "${PYTHON_EXECUTABLE}" -c "import sys; sys.stdout.write(';'.join([str(x) for x in sys.version_info[:3]]))"
                  OUTPUT_VARIABLE _VERSION
                  RESULT_VARIABLE _PYTHON_VERSION_RESULT
                  ERROR_QUIET)

  message(STATUS "PYTHON VERSION=${_VERSION}")

  # We will also install python in the install bin dir
  include(GNUInstallDirs)
  set( EMBEDDED_PYTHON_INSTALL_DIR ${CMAKE_INSTALL_BINDIR}/.. CACHE INTERNAL "" FORCE )
  install( DIRECTORY "${SOURCE_PYTHON_DIR}/" # trailing slash to avoid having an extra folder level in the copy
          DESTINATION "${EMBEDDED_PYTHON_INSTALL_DIR}"
          PATTERN "*.pyc" EXCLUDE
          PATTERN "include" EXCLUDE
          PATTERN "libs" EXCLUDE
          PATTERN "README.txt" EXCLUDE
          PATTERN "NEWS.txt" EXCLUDE
          PATTERN "LICENSE.txt" EXCLUDE
          PATTERN "bin/python*" EXCLUDE )

  # Copy the python executable with exec rights
  install( FILES "${SOURCE_PYTHON_DIR}/bin/python"
           DESTINATION "${EMBEDDED_PYTHON_INSTALL_DIR}/bin"
           PERMISSIONS OWNER_EXECUTE OWNER_WRITE OWNER_READ )

  install( FILES "${SOURCE_PYTHON_DIR}/bin/python2"
           DESTINATION "${EMBEDDED_PYTHON_INSTALL_DIR}/bin"
           PERMISSIONS OWNER_EXECUTE OWNER_WRITE OWNER_READ )

  if( ${CMAKE_VERSION} VERSION_LESS "3.12" )
    find_package( PythonInterp ${REQUIRED_PYTHON_VERSION} REQUIRED )
    # find_package( PythonLibs ${REQUIRED_PYTHON_VERSION} REQUIRED )
    unset(PYTHON_LIBRARY CACHE)
    unset(PYTHON_INCLUDE_DIR CACHE)

    set( PYTHON_LIBRARY "${EMBEDDED_PYTHON_DIR}/lib/libpython${REQUIRED_PYTHON_VERSION}.so" CACHE INTERNAL "" FORCE )
    if( NOT EXISTS ${PYTHON_LIBRARY} )
      message( FATAL_ERROR "The embedded python library was not found at ${EMBEDDED_PYTHON_DIR}/lib/libpython${REQUIRED_PYTHON_VERSION}.so" )
      return()
    else()
      message(STATUS "Set PYTHON_LIBRARY=${PYTHON_LIBRARY}")
      set(PYTHONLIBS_FOUND True)
    endif()

    set( PYTHON_INCLUDE_DIR "${EMBEDDED_PYTHON_DIR}/include/python${REQUIRED_PYTHON_VERSION}" CACHE INTERNAL "" FORCE )
    if( NOT EXISTS ${PYTHON_LIBRARY} )
      message( FATAL_ERROR "The embedded python include dir was not found at ${PYTHON_INCLUDE_DIR}/include/python${REQUIRED_PYTHON_VERSION}" )
      return()
    else()
      message(STATUS "Set PYTHON_INCLUDE_DIR=${PYTHON_INCLUDE_DIR}")
    endif()

  else()
    # CMake >= 3.12: find_package(PythonInterp ...) is deprecated
    set( Python_ROOT_DIR ${EMBEDDED_PYTHON_DIR} CACHE INTERNAL "" FORCE )
    set( Python2_ROOT_DIR ${EMBEDDED_PYTHON_DIR} CACHE INTERNAL "" FORCE )
    find_package( Python2 COMPONENTS Interpreter Development REQUIRED )
  endif()

endif()

if ( EMBEDDED_PYTHON AND ( DEFINED ENV{PYTHONHOME} OR DEFINED ENV{PYTHONPATH} ) )
  # When embedding a full python distribution, if a PYTHONHOME and/or PYTHONPATH env var
  # is defined, the packages we need may not always be installed at the correct place,
  # or pip could detect that some of them are already installed and not install them in our
  # local environment
  if( DEFINED ENV{PYTHONHOME} )
    message( FATAL_ERROR "A PYTHONHOME env var was detected (with value '$ENV{PYTHONHOME}'). Please remove any PYTHONHOME or PYTHONPATH env var first !" )
  endif()
  if(DEFINED ENV{PYTHONPATH} )
    message( FATAL_ERROR "A PYTHONPATH env var was detected (with value '$ENV{PYTHONPATH}'). Please remove any PYTHONHOME or PYTHONPATH env var first !" )
  endif()
endif()
