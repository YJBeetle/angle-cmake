# Copyright 2020 The SwiftShader Authors. All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

function(InitSubmodule target submodule_dir)
    if (NOT TARGET ${target})
        if(NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${submodule_dir}/.git)
            message(WARNING "
        Target ${target} from submodule ${submodule_dir} missing.
        Running 'git submodule update --init' to download it:
            ")

            execute_process(COMMAND git -C ${CMAKE_CURRENT_SOURCE_DIR} submodule update --init ${submodule_dir})
        endif()
    endif()
endfunction()

InitSubmodule(angle angle)

InitSubmodule(depot_tools depot_tools)

# angle setup

find_package(Python2 COMPONENTS Interpreter REQUIRED)

# Depend on .gclient file. Setup only runs if this file doesn't exist in the angle
# submodule checkout.
set(ANGLE_SETUP_OUTPUT
    "${CMAKE_CURRENT_SOURCE_DIR}/angle/.gclient"
)

if (WIN32)
    set(env ${CMAKE_COMMAND} -E env "PATH=${CMAKE_CURRENT_SOURCE_DIR}/depot_tools;$ENV{PATH}")
else()
    set(env env "PATH=${CMAKE_CURRENT_SOURCE_DIR}/depot_tools:$ENV{PATH}")
endif()

if (APPLE)
    set(sed_change2dot sed -i .bak s/change2dot/./ .gclient)
else()
    set(sed_change2dot sed -i s/change2dot/./ .gclient)
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    # On Linux, run install deps script
    set(linux_install_build_deps build/install-build-deps.sh --no-arm --no-nacl --no-chromeos-fonts)
endif()

add_custom_command(
    COMMENT "Setting up angle"
    OUTPUT ${ANGLE_SETUP_OUTPUT}
    VERBATIM
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/angle

    COMMAND ${env} gclient config --name change2dot --unmanaged https://chromium.googlesource.com/angle/angle.git
    COMMAND ${sed_change2dot}
    COMMAND ${env} gclient sync
    COMMAND ${linux_install_build_deps}
)
add_custom_target(angle-setup DEPENDS
    ${ANGLE_SETUP_OUTPUT}
)
set_target_properties(angle-setup PROPERTIES FOLDER ANGLE)


# angle build

# Depend on all angle source files, even though most of these aren't used to build
# the ouput files we want. Ninja will run very quickly anyway, and the output files
# are always touched to make sure subsequent builds do nothing.
file(GLOB_RECURSE ANGLE_SRC_FILES
    CONFIGURE_DEPENDS angle/src/*.cpp
)

set(ANGLE_BUILD_OUTPUT
  "${CMAKE_BINARY_DIR}/bin-angle/libEGL${CMAKE_SHARED_LIBRARY_SUFFIX}"
  "${CMAKE_BINARY_DIR}/bin-angle/libGLESv2${CMAKE_SHARED_LIBRARY_SUFFIX}"
)

# The root CMakeLists adds a dependency from angle onto the PowerVR GL example
# targets. For angle, we copy the PVR output 'bin' folder to 'bin-angle' into
# which we build the angle targets we want.
set(PVR_OUTPUT_DIR "${CMAKE_BINARY_DIR}/bin")
set(ANGLE_OUTPUT_DIR "${CMAKE_BINARY_DIR}/bin-angle")

# For some reason, "gn gen" returns a non-zero code on Windows, despite completing correctly,
# which makes the custom_command below not complete. For now, we work around this by wrapping
# up the call in a generated batch script.
if(WIN32)
    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/run_gn.bat "gn %*")
    set(GN_COMMAND ${CMAKE_CURRENT_BINARY_DIR}/run_gn.bat)
else()
    set(GN_COMMAND "gn")
endif()

add_custom_command(
    COMMENT "Building angle"
    OUTPUT ${ANGLE_BUILD_OUTPUT}
    DEPENDS ${ANGLE_SRC_FILES}
    VERBATIM
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/angle

    # Copy PowerVR's bin directory to bin-angle. Always create the bin folder in case PowerVR isn't being built.
    COMMAND ${CMAKE_COMMAND} -E make_directory ${PVR_OUTPUT_DIR}
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${PVR_OUTPUT_DIR} ${ANGLE_OUTPUT_DIR}
    # Delete the output files that may have been copied over. These are the SwiftShader versions, and we want the ANGLE ones.
    COMMAND ${CMAKE_COMMAND} -E remove -f ${ANGLE_BUILD_OUTPUT}

    # gn build angle targets we want
    COMMAND ${env} ${GN_COMMAND} gen ${ANGLE_OUTPUT_DIR}
    COMMAND ${env} autoninja -C ${ANGLE_OUTPUT_DIR} libEGL libGLESv2

    # Always update time stamps of output files since dependencies of this command (${ANGLE_SRC_FILES})
    # include files that aren't actually used to build the output files.
    COMMAND ${CMAKE_COMMAND} -E touch_nocreate ${ANGLE_BUILD_OUTPUT}
)
add_custom_target(angle ALL DEPENDS
    angle-setup
    ${ANGLE_BUILD_OUTPUT}
)
set_target_properties(angle PROPERTIES FOLDER ANGLE)

# Copy script to setup swangle environment to bin-angle
if(WIN32)
    set(EXPORT_SWANGLE_ENV_SCRIPT "export-swangle-env.bat")
else()
    set(EXPORT_SWANGLE_ENV_SCRIPT "export-swangle-env.sh")
endif()
configure_file(cmake/${EXPORT_SWANGLE_ENV_SCRIPT} ${ANGLE_OUTPUT_DIR}/${EXPORT_SWANGLE_ENV_SCRIPT})
set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS "${ANGLE_OUTPUT_DIR}/${EXPORT_SWANGLE_ENV_SCRIPT}")
