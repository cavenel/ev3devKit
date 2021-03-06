project ("ev3devkit" C)

if (CMAKE_CROSSCOMPILING)
    # we are depending on CMAKE_SYSROOT feature introduce in CMake 3.x
    # and we only support cross compiling using Ubuntu 16.04
    cmake_minimum_required (VERSION 3.5)
else (CMAKE_CROSSCOMPILING)
    cmake_minimum_required (VERSION 2.6)
endif (CMAKE_CROSSCOMPILING)

get_directory_property(PARENT_DIR PARENT_DIRECTORY)

if (NOT PARENT_DIR)
    option (EV3DEVKIT_BUILD_DEMO "Build the demo programs" Yes)
endif (NOT PARENT_DIR)

list (APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

set (DEVICES_DEMO_EXEC_NAME "ev3devKit-devices-demo")
set (UI_DEMO_EXEC_NAME "ev3devKit-ui-demo")
set (SOVERSION 0)
set (VERSION "0.5.2")
set (GIR_NAME "Ev3devKit")
set (GIR_VERSION "0.5")
set (EV3DEVKIT_PACKAGE ${PROJECT_NAME}-${GIR_VERSION})
set (LIBRARY_NAME "${EV3DEVKIT_PACKAGE}")
set (TARGET_GLIB "2.40")

if (PARENT_DIR)
    # If this is included by a parent project, export some variables back to the parent
    set (${PROJECT_NAME}_PACKAGE ${EV3DEVKIT_PACKAGE} PARENT_SCOPE)
    set (${PROJECT_NAME}_VAPI ${EV3DEVKIT_PACKAGE}.vapi PARENT_SCOPE)
endif (PARENT_DIR)

add_subdirectory (data)
set (XDG_DATA_DIR data)

# Comment this out to enable C compiler warnings
add_definitions (-w)

set (COMMON_PACKAGES
    gio-2.0
    gio-unix-2.0
    glib-2.0
    gmodule-2.0
    gobject-2.0
    grx-3.0
    gudev-1.0
    linux
    posix
)
set (COMMON_MODULES
    gio-2.0
    gio-unix-2.0
    glib-2.0
    gmodule-2.0
    gobject-2.0
    grx-3.0
    gudev-1.0
)

configure_file (src/Version.vala.in Version.vala)
# convert CMake list to space separated string for pkgconfig file
string(REPLACE ";" " " PKGCONFIG_DEPS "${COMMON_MODULES}")
configure_file (${PROJECT_NAME}.pc.in ${PROJECT_NAME}-${GIR_VERSION}.pc @ONLY)

set (LIBRARY_SOURCE_FILES
    src/devices/Cpu.vala
    src/devices/DcMotor.vala
    src/devices/Device.vala
    src/devices/DeviceManager.vala
    src/devices/Input.vala
    src/devices/Led.vala
    src/devices/MotorPolarity.vala
    src/devices/MotorStateFlags.vala
    src/devices/Port.vala
    src/devices/PowerSupply.vala
    src/devices/Sensor.vala
    src/devices/ServoMotor.vala
    src/devices/TachoMotor.vala
    src/ui/Box.vala
    src/ui/Button.vala
    src/ui/CheckboxMenuItem.vala
    src/ui/CheckButton.vala
    src/ui/Container.vala
    src/ui/Dialog.vala
    src/ui/Fonts.vala
    src/ui/Grid.vala
    src/ui/Icon.vala
    src/ui/InputDialog.vala
    src/ui/Label.vala
    src/ui/Menu.vala
    src/ui/MenuItem.vala
    src/ui/MessageDialog.vala
    src/ui/OnScreenKeyboard.vala
    src/ui/RadioMenuItem.vala
    src/ui/Rectangle.vala
    src/ui/Screen.vala
    src/ui/Scroll.vala
    src/ui/Spacer.vala
    src/ui/Stack.vala
    src/ui/StatusBar.vala
    src/ui/StatusBarItem.vala
    src/ui/TextEntry.vala
    src/ui/Widget.vala
    src/ui/Window.vala
    src/ConsoleApp.vala
    ${CMAKE_CURRENT_BINARY_DIR}/Version.vala
)

set (DEVICES_DEMO_SOURCE_FILES
    demo/devices_demo.vala
)

set (UI_DEMO_COMMON_SOURCE_FILES
    demo/UiDemoWindow.vala
)

set (UI_DEMO_SOURCE_FILES
    demo/ui_demo.vala
    ${UI_DEMO_COMMON_SOURCE_FILES}
)

find_package (PkgConfig REQUIRED)

pkg_check_modules (DEPS REQUIRED ${COMMON_MODULES})
add_definitions (${DEPS_CFLAGS})

find_package (Vala REQUIRED)
include (ValaVersion)
ensure_vala_version ("0.24" MINIMUM)
include (ValaPrecompile)

find_package(GirCompiler REQUIRED)

vala_precompile (LIBRARY_VALA_C ${LIBRARY_NAME}
LIBRARY
    ${LIBRARY_SOURCE_FILES}
PACKAGES
    ${COMMON_PACKAGES}
CUSTOM_VAPIS
    bindings/*.vapi
OPTIONS
    --target-glib=${TARGET_GLIB}
    --thread
GENERATE_VAPI
    ${EV3DEVKIT_PACKAGE}
GENERATE_HEADER
    ${EV3DEVKIT_PACKAGE}
GENERATE_GIR
    ${GIR_NAME}-buggy-${GIR_VERSION}
)

# Work around bug in valac where .gir is missing the "shared-library" attribute
# in the namespace node. This is needed by dh_girepository to generate the
# {shlibs:Depends} variable.

add_custom_command (
    OUTPUT
        "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}-${GIR_VERSION}.gir"
    COMMAND
        "${CMAKE_CURRENT_SOURCE_DIR}/fix-gir.sh"
    ARGS
        "$<TARGET_SONAME_FILE_NAME:${LIBRARY_NAME}-shared>"
        "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}-buggy-${GIR_VERSION}.gir"
        "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}-${GIR_VERSION}.gir"
    DEPENDS
        "${CMAKE_CURRENT_SOURCE_DIR}/fix-gir.sh"
        "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}-buggy-${GIR_VERSION}.gir"
    COMMENT
        "Fixing ${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}-buggy-${GIR_VERSION}.gir"
)

add_custom_command (
    OUTPUT
        "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}-${GIR_VERSION}.typelib"
    COMMAND
        ${G_IR_COMPILER_EXECUTABLE}
    ARGS
        "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}-${GIR_VERSION}.gir"
        "--shared-library=$<TARGET_SONAME_FILE_NAME:${LIBRARY_NAME}-shared>"
        "--output=${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}-${GIR_VERSION}.typelib"
        "--includedir=${CMAKE_CURRENT_BINARY_DIR}"
    DEPENDS
        "${LIBRARY_NAME}-shared"
        "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}-${GIR_VERSION}.gir"
    COMMENT
        "Genterating typelib."
)

add_custom_target ("${LIBRARY_NAME}-typelib"
    ALL
    DEPENDS
        "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}-${GIR_VERSION}.typelib"
)

# end of workaround

add_library(${LIBRARY_NAME} OBJECT ${LIBRARY_VALA_C})
set_target_properties(${LIBRARY_NAME} PROPERTIES POSITION_INDEPENDENT_CODE 1)
add_library(${LIBRARY_NAME}-shared SHARED $<TARGET_OBJECTS:${LIBRARY_NAME}>)
target_link_libraries(${LIBRARY_NAME}-shared ${DEPS_LIBRARIES})
set_target_properties(${LIBRARY_NAME}-shared PROPERTIES
    VERSION "${VERSION}"
    SOVERSION "${SOVERSION}"
    OUTPUT_NAME ${LIBRARY_NAME})
add_library(${LIBRARY_NAME}-static STATIC $<TARGET_OBJECTS:${LIBRARY_NAME}>)
target_link_libraries(${LIBRARY_NAME}-static ${DEPS_LIBRARIES})
set_target_properties(${LIBRARY_NAME}-static PROPERTIES
    OUTPUT_NAME ${LIBRARY_NAME})
add_dependencies(${LIBRARY_NAME} ev3devkit_data)
install (TARGETS ${LIBRARY_NAME}-shared LIBRARY DESTINATION lib)
install (TARGETS ${LIBRARY_NAME}-static ARCHIVE DESTINATION lib)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${EV3DEVKIT_PACKAGE}.h DESTINATION include/)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-${GIR_VERSION}.pc DESTINATION lib/pkgconfig/)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${EV3DEVKIT_PACKAGE}.vapi DESTINATION share/vala/vapi/)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}-${GIR_VERSION}.gir DESTINATION share/gir-1.0/)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}-${GIR_VERSION}.typelib DESTINATION lib/girepository-1.0/)

if (EV3DEVKIT_BUILD_DEMO)
    set (EXEC_PACKAGES ${COMMON_PACKAGES} ${EV3DEVKIT_PACKAGE})
    set (EXEC_LIBRARY_DEPENDS ${LIBRARY_NAME}-shared)

    # Device driver demo program

    vala_precompile(DEVICES_DEMO_EXEC_VALA_C ${DEVICES_DEMO_EXEC_NAME}
        ${DEVICES_DEMO_SOURCE_FILES}
    PACKAGES
        ${EXEC_PACKAGES}
    CUSTOM_VAPIS
        bindings/*.vapi
    OPTIONS
        --target-glib=${TARGET_GLIB}
        --thread
        --vapidir=${CMAKE_CURRENT_BINARY_DIR}
    )

    add_executable (${DEVICES_DEMO_EXEC_NAME} ${DEVICES_DEMO_EXEC_VALA_C})
    target_link_libraries (${DEVICES_DEMO_EXEC_NAME} ${EXEC_LIBRARY_DEPENDS})
    if (CMAKE_CROSSCOMPILING)
        # These are not picked up automatically when cross compiling for some reason
        target_link_libraries (${DEVICES_DEMO_EXEC_NAME} "-pthread" "-lresolv")
    endif (CMAKE_CROSSCOMPILING)

    add_custom_target (run-${DEVICES_DEMO_EXEC_NAME}
        COMMAND
            XDG_DATA_DIRS=${XDG_DATA_DIR}:/usr/local/share:/usr/share
            ./${DEVICES_DEMO_EXEC_NAME}
        DEPENDS ${DEVICES_DEMO_EXEC_NAME}
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )

    install (TARGETS ${DEVICES_DEMO_EXEC_NAME} RUNTIME DESTINATION bin)


    # User interface demo program

    vala_precompile(UI_DEMO_EXEC_VALA_C ${UI_DEMO_EXEC_NAME}
        ${UI_DEMO_SOURCE_FILES}
    PACKAGES
        ${EXEC_PACKAGES}
    CUSTOM_VAPIS
        bindings/*.vapi
    OPTIONS
        --target-glib=${TARGET_GLIB}
        --thread
        --vapidir=${CMAKE_CURRENT_BINARY_DIR}
    )

    add_executable (${UI_DEMO_EXEC_NAME} ${UI_DEMO_EXEC_VALA_C})
    target_link_libraries (${UI_DEMO_EXEC_NAME} ${EXEC_LIBRARY_DEPENDS})
    if (CMAKE_CROSSCOMPILING)
        # These are not picked up automatically when cross compiling for some reason
        target_link_libraries (${UI_DEMO_EXEC_NAME} "-pthread" "-lresolv")
    endif (CMAKE_CROSSCOMPILING)

    add_custom_target (run-${UI_DEMO_EXEC_NAME}
        COMMAND
            XDG_DATA_DIRS=${XDG_DATA_DIR}:/usr/local/share:/usr/share
            ./${UI_DEMO_EXEC_NAME}
        DEPENDS ${UI_DEMO_EXEC_NAME}
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )

    install (TARGETS ${UI_DEMO_EXEC_NAME} RUNTIME DESTINATION bin)

endif (EV3DEVKIT_BUILD_DEMO)

find_package (Valadoc)
if (VALADOC_FOUND)
    include (ValadocGenerate)
    generate_valadoc (
        ${LIBRARY_SOURCE_FILES}
        bindings/*.vapi
    PACKAGE_NAME
        ${PROJECT_NAME}
    PACKAGE_VERSION
        ${GIR_VERSION}
    OPTIONS
        --target-glib=${TARGET_GLIB}
    PACKAGES
        ${COMMON_PACKAGES}
    )
    add_custom_target (doc ALL DEPENDS valadoc)
endif (VALADOC_FOUND)
