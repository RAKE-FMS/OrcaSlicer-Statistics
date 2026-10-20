get_filename_component(_TEST_NAME ${CMAKE_CURRENT_LIST_DIR} NAME)
add_executable(${_TEST_NAME}_tests
    ${_TEST_NAME}_tests_main.cpp
    test_preview_capture.cpp
    )

if (MSVC)
    target_link_libraries(${_TEST_NAME}_tests Setupapi.lib)
endif ()

target_link_libraries(${_TEST_NAME}_tests test_common libslic3r_gui libslic3r Catch2::Catch2WithMain)
# FindwxWidgets on Linux exposes directory settings instead of imported target usage requirements.
get_directory_property(_wx_includes DIRECTORY ${PROJECT_SOURCE_DIR}/src DEFINITION wxWidgets_INCLUDE_DIRS)
get_directory_property(_wx_definitions DIRECTORY ${PROJECT_SOURCE_DIR}/src DEFINITION wxWidgets_DEFINITIONS)
get_directory_property(_wx_cxx_flags DIRECTORY ${PROJECT_SOURCE_DIR}/src DEFINITION wxWidgets_CXX_FLAGS)
separate_arguments(_wx_cxx_flags NATIVE_COMMAND "${_wx_cxx_flags}")
target_include_directories(${_TEST_NAME}_tests SYSTEM PRIVATE ${_wx_includes})
target_compile_definitions(${_TEST_NAME}_tests PRIVATE ${_wx_definitions})
target_compile_options(${_TEST_NAME}_tests PRIVATE ${_wx_cxx_flags})
set_property(TARGET ${_TEST_NAME}_tests PROPERTY FOLDER "tests")

orcaslicer_copy_test_dlls()

orcaslicer_discover_tests(${_TEST_NAME}_tests)
