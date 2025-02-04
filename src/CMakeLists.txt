CMAKE_MINIMUM_REQUIRED(VERSION 3.14 FATAL_ERROR)
set (CMAKE_CXX_STANDARD 17)

file(GLOB HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/*.hpp)
source_group("SrcHeaders" FILES ${HEADERS})

include_directories(
  ${INCLUDE_DIRECTORIES}
  ${relic_SOURCE_DIR}/include
  ${relic_BINARY_DIR}/include
  )

if (GMP_FOUND)
  include_directories(${GMP_INCLUDES})
endif(GMP_FOUND)

set(C_LIB ${CMAKE_BINARY_DIR}/libbls.a)

add_library(bls ${CMAKE_CURRENT_SOURCE_DIR}/privatekey.cpp)

add_library(blstmp ${HEADERS}
  ${CMAKE_CURRENT_SOURCE_DIR}/privatekey.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/bls.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/elements.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/schemes.cpp
)

target_compile_definitions(blstmp PRIVATE BLSALLOC_SODIUM=1)
target_link_libraries(blstmp PUBLIC sodium)

if(MSVC)
  add_custom_command(
    OUTPUT ${C_LIB}
    COMMAND ${CMAKE_AR} /OUT:${C_LIB} $<TARGET_FILE:blstmp> $<TARGET_FILE:relic_s>
    DEPENDS blstmp relic_s
  )
  add_custom_target(combined_custom
    DEPENDS ${C_LIB}
  )
else()
  set(OPREFIX object_)
  set(LIBRARIES_TO_COMBINE
    COMMAND ${CMAKE_COMMAND} -E make_directory ${OPREFIX}$<TARGET_NAME:blstmp> && cd ${OPREFIX}$<TARGET_NAME:blstmp> && ${CMAKE_AR} -x $<TARGET_FILE:blstmp>
    COMMAND ${CMAKE_COMMAND} -E make_directory ${OPREFIX}$<TARGET_NAME:relic_s> && cd ${OPREFIX}$<TARGET_NAME:relic_s> && ${CMAKE_AR} -x $<TARGET_FILE:relic_s>
  )

  find_library(GMP_NAME NAMES libgmp.a)
  if(NOT ${GMP_NAME} STREQUAL "GMP_NAME-NOTFOUND")
    list(APPEND LIBRARIES_TO_COMBINE COMMAND ${CMAKE_COMMAND} -E make_directory ${OPREFIX}gmp && cd ${OPREFIX}gmp && ${CMAKE_AR} -x ${GMP_NAME})
  endif()

  add_custom_target(combined_custom
    ${LIBRARIES_TO_COMBINE}
    COMMAND ${CMAKE_AR} -rs ${C_LIB} ${OPREFIX}*/*${CMAKE_C_OUTPUT_EXTENSION}
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS blstmp relic_s
  )
endif()

add_library(combined STATIC IMPORTED GLOBAL)
add_dependencies(combined combined_custom)
target_link_libraries(bls combined)

set_target_properties(combined
  PROPERTIES
  IMPORTED_LOCATION ${C_LIB}
)

file(GLOB includes "${CMAKE_CURRENT_SOURCE_DIR}/*.hpp")
install(DIRECTORY ${relic_SOURCE_DIR}/include/ DESTINATION include/chiabls)
install(DIRECTORY ${relic_BINARY_DIR}/include/ DESTINATION include/chiabls)
install(FILES ${includes} DESTINATION include/chiabls)
install(FILES ${C_LIB} DESTINATION lib)

if(BUILD_BLS_TESTS)
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../contrib/catch)
  add_executable(runtest test.cpp)
  target_link_libraries(runtest blstmp relic_s)
endif()

if(BUILD_BLS_BENCHMARKS)
  add_executable(runbench test-bench.cpp)
  target_link_libraries(runbench blstmp relic_s)
endif()
