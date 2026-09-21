cmake_minimum_required(VERSION 3.14)

project(pybind11_bindings_generator)

# userdefined variables
set(headers_to_bind_file "" CACHE PATH "file containing the headers to be bound")
set(root_module_name "bound_project" CACHE STRING "name of the generated python package")
set(project_lib_file "" CACHE PATH "location of the library file for the project to bind")
set(project_includes "" CACHE STRING "space seperated list of additional includes")
set(project_extra_sources "" CACHE STRING "extra sources to build as space seperated list")
set(output_dir "${CMAKE_CURRENT_BINARY_DIR}" CACHE PATH "folder to output files, usually the binary folder of the project using this cmake file")
set(compile_commands_file "" CACHE PATH "path to compile_commands.json")
set(binder_config_file "" CACHE PATH "path to a binder compatible config file")

STRING(REGEX REPLACE " " ";" project_includes "${project_includes}")
STRING(REGEX REPLACE " " ";" project_extra_sources "${project_extra_sources}")

add_library(bound_project SHARED IMPORTED)
set_target_properties(
	bound_project 
	PROPERTIES 
	IMPORTED_LOCATION ${project_lib_file}
)
# get bound_project's defines:
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/getDefines.sh ${CMAKE_CURRENT_BINARY_DIR}/getDefines.sh @ONLY)
execute_process(
	COMMAND sh ${CMAKE_CURRENT_BINARY_DIR}/getDefines.sh
	OUTPUT_VARIABLE PROJECT_DEFINES
)
# STRING(REGEX REPLACE ";" "\\\\;" PROJECT_DEFINES "${PROJECT_DEFINES}")
STRING(REGEX REPLACE "\n" ";" PROJECT_DEFINES "${PROJECT_DEFINES}")
foreach(DEFINE ${PROJECT_DEFINES})
	add_definitions(${DEFINE})
endforeach()


set(Python_FIND_VIRTUALENV "FIRST")
set(Python_FIND_STRATEGY "LOCATION")
find_package(Python 3.0 COMPONENTS Interpreter Development NumPy REQUIRED)

# try finding pybind11, otherwise build it from source:
find_package(pybind11)
if(NOT pybind11_FOUND)
	if(NOT pybind11_POPULATED)
		include(FetchContent)
		FetchContent_Declare(
			pybind11
			# RosettaCommons provide their own pybind11 fork which is guaranteed to work
			GIT_REPOSITORY https://github.com/RosettaCommons/pybind11.git
		)
		
		# set CMAKE_ARGS before add_subdirectory per set:
		set(PYBIND11_TEST OFF CACHE BOOL "")
		
		FetchContent_MakeAvailable(pybind11)
	endif()
endif()

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/createBindings.sh ${CMAKE_CURRENT_BINARY_DIR}/createBindings.sh @ONLY)
configure_file(${binder_config_file} ${CMAKE_CURRENT_BINARY_DIR}/binder.cfg @ONLY)

# generate bindings:
execute_process(
	COMMAND sh ${CMAKE_CURRENT_BINARY_DIR}/createBindings.sh
)

file(READ ${CMAKE_CURRENT_BINARY_DIR}/bindings/${root_module_name}.sources SOURCES)

# found this technique to build the cmake list here:
# http://public.kitware.com/pipermail/cmake/2007-May/014236.html
STRING(REGEX REPLACE ";" "\\\\;" SOURCES "${SOURCES}")
STRING(REGEX REPLACE "\n" ";" SOURCES "${SOURCES}")
list(TRANSFORM SOURCES PREPEND ${CMAKE_CURRENT_BINARY_DIR}/bindings/)

# add pybind modules:
pybind11_add_module(
	${root_module_name} SHARED
	${SOURCES}
	${project_extra_sources}
)

target_include_directories(
	${root_module_name} PUBLIC
	pybind11::headers
	${Python_NumPy_INCLUDE_DIRS}
	${project_includes}
)
target_link_libraries(
	${root_module_name} PUBLIC
	${Python_LIBRARIES}
	pybind11::module
	bound_project
)
set_target_properties(
	${root_module_name} 
	PROPERTIES 
	LIBRARY_OUTPUT_DIRECTORY "${output_dir}"
	OUTPUT_NAME "${root_module_name}"
	PREFIX "" SUFFIX ".so"
)