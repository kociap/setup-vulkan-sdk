# CMAKE VARIABLES:
#   VULKAN_SDK -- where to install SDK components
#   VULKAN_SDK_CONFIG -- config.json that specifies repos/branches
#   VULKAN_SDK_COMPONENTS -- SDK components to install
#
# A Vulkan SDK config.json may be downloaded using the template:
#   https://vulkan.lunarg.com/sdk/config/VERSION/PLATFORM/config.json
# See: https://vulkan.lunarg.com/content/view/latest-sdk-version-api
#
# For a list of available components, see a particular Vulkan SDK release.

# NOTE: 3.19+ needed for JSON parsing
cmake_minimum_required(VERSION 3.19)

include(FetchContent)
include(cmake/json.cmake)

if(NOT VULKAN_SDK)
  message(FATAL_ERROR
    "Provide the Vulkan SDK install directory -DVULKAN_SDK=<path>")
endif()
if(NOT VULKAN_SDK_CONFIG)
  message("Provide Vulkan SDK config file via -DVULKAN_SDK_CONFIG=<file>")
endif()
if(NOT VULKAN_SDK_COMPONENTS)
  message(FATAL_ERROR
    "Provide components -DVULKAN_SDK_COMPONENTS=<comma-separated list>")
endif()

project(VulkanSDKComponents)

function(setup_component PACKAGE_NAME)
  if (TARGET ${PACKAGE_NAME})
    return()
  endif()

  json_get(PACKAGE_URL ${VULKAN_SDK_CONFIG_CONTENT} "repos" ${PACKAGE_NAME} "url")
  if(NOT PACKAGE_URL)
    message(FATAL_ERROR "Could not read URL of ${PACKAGE_NAME}")
  endif()
  json_get(PACKAGE_DEPENDENCIES ${VULKAN_SDK_CONFIG_CONTENT} "repos" ${PACKAGE_NAME} "dependencies")
  # TODO: Fix this function
  json_coalesce_subprops("${VULKAN_SDK_CONFIG_CONTENT}" ref "repos" "${PACKAGE_NAME}" PROPERTIES commit tag branch)
  set(PACKAGE_VERSION ${ref})

  # Process config.json dependencies.
  json_foreach("${PACKAGE_DEPENDENCIES}" "" setup_component ${PACKAGE_NAME})

  message(STATUS "Fetching ${PACKAGE_NAME} ${PACKAGE_VERSION} from ${PACKAGE_URL}")
  FetchContent_Declare(${PACKAGE_NAME}
    GIT_REPOSITORY "${PACKAGE_URL}"
    GIT_TAG "${PACKAGE_VERSION}"
  )
  FetchContent_MakeAvailable(${PACKAGE_NAME})
  FetchContent_GetProperties(${PACKAGE_NAME}
    SOURCE_DIR PACKAGE_SOURCE_DIRECTORY
    BINARY_DIR PACKAGE_BINARY_DIRECTORY
  )

  # Some of the projects are built weirdly, i.e. we download a repository
  # containing a python script which downloads the source and its dependencies.
  # This is only applicable to components which have a "script" property.
  json_get(PACKAGE_SCRIPT ${VULKAN_SDK_CONFIG_CONTENT} "repos" ${PACKAGE_NAME} "script")
  if(PACKAGE_SCRIPT)
    message(STATUS
      "Running script for ${PACKAGE_NAME}. Dependencies will be downloaded by the script.")
    # The script strings stored in an array are of the form
    #   python filename.py
    # We want the filename, hence we convert the string to a list.
    json_get(SCRIPT_STRING ${PACKAGE_SCRIPT} 0)
    string(REPLACE " " ";" SCRIPT_ARGS ${SCRIPT_STRING})
    list(GET SCRIPT_ARGS 1 SCRIPT_FILENAME)
    # Run the python script. By default it stores the downloaded source in the
    # 'src' subdirectory within the directory of the package.
    execute_process(COMMAND python ${SCRIPT_FILENAME}
                    WORKING_DIRECTORY ${PACKAGE_SOURCE_DIRECTORY}
                    OUTPUT_QUIET)
    add_subdirectory("${PACKAGE_SOURCE_DIRECTORY}/src" "${PACKAGE_BINARY_DIRECTORY}")
  endif()
endfunction()

################################################################################

set(CMAKE_INSTALL_PREFIX "${VULKAN_SDK}" CACHE PATH "Install prefix" FORCE)
message(STATUS "Install directory set to CMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}")

file(READ ${VULKAN_SDK_CONFIG} VULKAN_SDK_CONFIG_CONTENT)

string(REGEX REPLACE "[, ]+" ";" VULKAN_SDK_COMPONENTS "${VULKAN_SDK_COMPONENTS}")
list(JOIN VULKAN_SDK_COMPONENTS " " REQUESTED_COMPONENTS)
json_get(VULKAN_SDK_VERSION ${VULKAN_SDK_CONFIG_CONTENT} "version")
json_get(VULKAN_SDK_RELEASE_DATE ${VULKAN_SDK_CONFIG_CONTENT} "release_date")
message("---------------------------------------------------------------------")
message("SDK COMPONENTS:  ${REQUESTED_COMPONENTS}")
message("    BUILD PATH:  ${CMAKE_BINARY_DIR}")
message("  INSTALL PATH:  ${VULKAN_SDK}")
message(" CONFIGURATION:  ${VULKAN_SDK_CONFIG}")
message("       VERSION:  ${VULKAN_SDK_VERSION}")
message("  RELEASE_DATE:  ${VULKAN_SDK_RELEASE_DATE}")
message("---------------------------------------------------------------------")

# C++ modules are not supported by make.
set(VULKAN_HEADERS_ENABLE_MODULE OFF)
set(VULKAN_HEADERS_ENABLE_INSTALL ON)
# Do not build additional shaderc targets.
set(SHADERC_SKIP_TESTS ON)
set(SHADERC_SKIP_EXAMPLES ON)
set(SHADERC_SKIP_COPYRIGHT_CHECK ON)
set(SHADERC_ENABLE_SHARED_CRT ON)

foreach(PACKAGE_NAME ${VULKAN_SDK_COMPONENTS})
  setup_component(${PACKAGE_NAME})
endforeach()

string(TIMESTAMP VULKAN_SDK_BUILD_DATE UTC)
file(WRITE ${CMAKE_BINARY_DIR}/sdk.env
  "\
VULKAN_SDK=${VULKAN_SDK}
VULKAN_SDK_VERSION=${VULKAN_SDK_VERSION}
VULKAN_SDK_BUILD_DATE=${VULKAN_SDK_BUILD_DATE}
VULKAN_SDK_RELEASE_DATE=${VULKAN_SDK_RELEASE_DATE}
VULKAN_SDK_CONFIG=${VULKAN_SDK_CONFIG}
VULKAN_SDK_COMPONENTS=\"${VULKAN_SDK_COMPONENTS}\"
")

install(FILES ${CMAKE_BINARY_DIR}/sdk.env DESTINATION .)
