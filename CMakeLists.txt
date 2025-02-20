set(PACKAGE_PROJECT_ROOT_PATH
    "${CMAKE_CURRENT_LIST_DIR}"
    CACHE INTERNAL "The path to the PackageProject directory"
)

function(packageProject)
  include(CMakePackageConfigHelpers)
  include(GNUInstallDirs)

  cmake_parse_arguments(
    PROJECT
    ""
    "NAME;VERSION;INCLUDE_DIR;INCLUDE_DESTINATION;BINARY_DIR;COMPATIBILITY;VERSION_HEADER;NAMESPACE;DISABLE_VERSION_SUFFIX"
    "DEPENDENCIES"
    ${ARGN}
  )

  # optional feature: TRUE or FALSE or UNDEFINED! These variables will then hold the respective
  # value from the argument list or be undefined if the associated one_value_keyword could not be
  # found.
  if(PROJECT_DISABLE_VERSION_SUFFIX)
    unset(PROJECT_VERSION_SUFFIX)
  else()
    set(PROJECT_VERSION_SUFFIX -${PROJECT_VERSION})
  endif()

  # handle default arguments:
  if(NOT DEFINED PROJECT_COMPATIBILITY)
    set(PROJECT_COMPATIBILITY AnyNewerVersion)
  endif()

  # we want to automatically add :: to our namespace, so only append if a namespace was given in the
  # first place we also provide an alias to ensure that local and installed versions have the same
  # name
  if(DEFINED PROJECT_NAMESPACE)
    set(PROJECT_NAMESPACE ${PROJECT_NAMESPACE}::)
    add_library(${PROJECT_NAMESPACE}${PROJECT_NAME} ALIAS ${PROJECT_NAME})
  endif()

  if(DEFINED PROJECT_VERSION_HEADER)
    set(PROJECT_VERSION_INCLUDE_DIR ${PROJECT_BINARY_DIR}/PackageProjectInclude)

    string(TOUPPER ${PROJECT_NAME} UPPERCASE_PROJECT_NAME)
    configure_file(
      ${PACKAGE_PROJECT_ROOT_PATH}/version.h.in
      ${PROJECT_VERSION_INCLUDE_DIR}/${PROJECT_VERSION_HEADER} @ONLY
    )

    get_target_property(target_type ${PROJECT_NAME} TYPE)
    if(target_type STREQUAL "INTERFACE_LIBRARY")
      set(VISIBILITY INTERFACE)
    else()
      set(VISIBILITY PUBLIC)
    endif()
    target_include_directories(
      ${PROJECT_NAME} ${VISIBILITY} "$<BUILD_INTERFACE:${PROJECT_VERSION_INCLUDE_DIR}>"
    )
    install(DIRECTORY ${PROJECT_VERSION_INCLUDE_DIR}/ DESTINATION ${PROJECT_INCLUDE_DESTINATION})
  endif()

  write_basic_package_version_file(
    "${PROJECT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY ${PROJECT_COMPATIBILITY}
  )

  install(
    TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}Targets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}/${PROJECT_NAME}${PROJECT_VERSION_SUFFIX}
            COMPONENT Runtime
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}/${PROJECT_NAME}${PROJECT_VERSION_SUFFIX}
            COMPONENT Development
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}/${PROJECT_NAME}${PROJECT_VERSION_SUFFIX}
            COMPONENT Runtime
    BUNDLE DESTINATION ${CMAKE_INSTALL_BINDIR}/${PROJECT_NAME}${PROJECT_VERSION_SUFFIX}
           COMPONENT Runtime
    PUBLIC_HEADER DESTINATION ${PROJECT_INCLUDE_DESTINATION} COMPONENT Development
  )

  configure_package_config_file(
    ${PACKAGE_PROJECT_ROOT_PATH}/Config.cmake.in "${PROJECT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    INSTALL_DESTINATION lib/cmake/${PROJECT_NAME}${PROJECT_VERSION_SUFFIX}
  )

  install(
    EXPORT ${PROJECT_NAME}Targets
    DESTINATION lib/cmake/${PROJECT_NAME}${PROJECT_VERSION_SUFFIX}
    NAMESPACE ${PROJECT_NAMESPACE}
  )

  install(FILES "${PROJECT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
                "${PROJECT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
          DESTINATION lib/cmake/${PROJECT_NAME}${PROJECT_VERSION_SUFFIX}
  )

  install(DIRECTORY ${PROJECT_INCLUDE_DIR}/ DESTINATION ${PROJECT_INCLUDE_DESTINATION})

  set(${PROJECT_NAME}_VERSION
      ${PROJECT_VERSION}
      CACHE INTERNAL ""
  )
endfunction()
