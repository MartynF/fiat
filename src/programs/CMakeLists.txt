# (C) Copyright 2020- ECMWF.
#
# This software is licensed under the terms of the Apache Licence Version 2.0
# which can be obtained at http://www.apache.org/licenses/LICENSE-2.0.
# In applying this licence, ECMWF does not waive the privileges and immunities
# granted to it by virtue of its status as an intergovernmental organisation
# nor does it submit to any jurisdiction.

### fiat information tool

set( tool fiat )

get_property( langs GLOBAL PROPERTY ENABLED_LANGUAGES )

foreach( lang ${langs} )
  set( EC_${lang}_FLAGS "${CMAKE_${lang}_FLAGS} ${CMAKE_${lang}_FLAGS_${CMAKE_BUILD_TYPE_CAPS}}" )
endforeach()

configure_file( ${tool}.in ${CMAKE_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/${tool} @ONLY )

file(COPY ${CMAKE_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/${tool}
  DESTINATION ${CMAKE_BINARY_DIR}/bin
  FILE_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ
  GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)

install(
  FILES  ${CMAKE_BINARY_DIR}/bin/${tool}
  DESTINATION   ${INSTALL_BIN_DIR}
  PERMISSIONS   OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
)

### drhook-merge tool

list( APPEND files
  drhook-merge
)

install(
  FILES       ${files}
  DESTINATION ${INSTALL_BIN_DIR}
  PERMISSIONS   OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
)
foreach( file ${files} )
  execute_process(COMMAND "${CMAKE_COMMAND}" "-E" "create_symlink"
      "${CMAKE_CURRENT_SOURCE_DIR}/${file}"
      "${CMAKE_BINARY_DIR}/${INSTALL_BIN_DIR}/${file}")
endforeach()


### Program fiat-printbinding
# Note: it only depends on MPI and OpenMP
ecbuild_add_executable(TARGET fiat-printbinding
  SOURCES fiat-printbinding.c )
find_package( OpenMP COMPONENTS C )
if( TARGET OpenMP::OpenMP_C )
  target_link_libraries( fiat-printbinding OpenMP::OpenMP_C )
endif()

find_package( MPI COMPONENTS C )
if( HAVE_MPI AND TARGET MPI::MPI_C )
  target_link_libraries( fiat-printbinding MPI::MPI_C )
else()
  target_compile_definitions( fiat-printbinding PRIVATE NOMPI )
endif()

# On LUMI with Cray compiler we have false warnings about implicit function declaration (CPU_ZERO, sched_affinity)
check_c_compiler_flag( "-Wno-implicit-function-declaration" disable_warning_implicit_function_declaration )
if( disable_warning_implicit_function_declaration )
  target_compile_options( fiat-printbinding PRIVATE "-Wno-implicit-function-declaration" )
endif()

### Program fiat-drhook-sanity
# Note: it only depends on MPI and OpenMP
ecbuild_add_executable(TARGET fiat-drhook-sanity
  SOURCES fiat-drhook-sanity.F90 fiat-drhook-sanity-stream.F90 fiat-drhook-sanity-gemm.F90 mysecond.c
  LIBS    fiat parkind_dp
  )
target_compile_definitions( fiat-drhook-sanity PRIVATE OMP )

ecbuild_info( CMAKE_Fortran_COMPILER_ID -  1. ${CMAKE_Fortran_COMPILER_ID} 2. ${EC_COMPILER_FAMILY}  )
if( ${CMAKE_Fortran_COMPILER_ID} MATCHES Intel )
  set_source_files_properties(fiat-drhook-sanity-stream.f90 PROPERTIES COMPILE_FLAGS " -qopt-prefetch-distance=64,12 -qopt-streaming-cache-evict=0 -qopt-streaming-stores always -qopt-zmm-usage=high ")
endif()

find_package( OpenMP COMPONENTS Fortran )
if( TARGET OpenMP::OpenMP_Fortran )
  target_link_libraries( fiat-drhook-sanity OpenMP::OpenMP_Fortran )
endif()
target_link_libraries( fiat-drhook-sanity ${MKL_LIBRARIES} )

find_package( MPI COMPONENTS C )
if( HAVE_MPI AND TARGET MPI::MPI_C )
  target_link_libraries( fiat-drhook-sanity MPI::MPI_C )
else()
  target_compile_definitions( fiat-drhook-sanity PRIVATE NOMPI )
endif()


