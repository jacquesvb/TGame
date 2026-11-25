cmake_minimum_required(VERSION 3.16)

set(CMAKE_CXX_STANDARD  17)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# ************************
# Set this to ON instead of OFF to ship the game
# This will change the RESOURCES_PATH to be the local path
# ************************
set(PRODUCTION_BUILD OFF CACHE BOOL "Make this a production build" FORCE)
# delete the "out" folder after changing if VSCode doesn't recognize the change


if(MSVC) 
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Release>:Release>")
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
add_compile_options(/arch:AVX2) #make sure SIMD optimizations take place
else()
	add_compile_options(-mavx2 -mfma) # make sure SIMD optimizations take place, but on linux and other platforms
endif()

project(mygame)

# ************************
# Add libraries here
# ************************

# raylib is vendored inside thirdparty/raylib. If it isn't there yet, it is
# cloned from GitHub during the first CMake configure and built along with the
# project, so no system-wide installation is required.
set(RAYLIB_VERSION "5.5" CACHE STRING "raylib version used when fetching")
set(RAYLIB_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/raylib")

# Make sure the thirdparty folder exists before checking for raylib inside it.
set(THIRDPARTY_DIR "${CMAKE_CURRENT_SOURCE_DIR}/thirdparty")
if(NOT EXISTS "${THIRDPARTY_DIR}")
	message(STATUS "thirdparty folder not found, creating ${THIRDPARTY_DIR}")
	file(MAKE_DIRECTORY "${THIRDPARTY_DIR}")
endif()

# Only build the raylib library itself, not its examples or games.
set(BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(BUILD_GAMES OFF CACHE BOOL "" FORCE)

if(EXISTS "${RAYLIB_SOURCE_DIR}/CMakeLists.txt")
	message(STATUS "raylib found at ${RAYLIB_SOURCE_DIR}")
	add_subdirectory("${RAYLIB_SOURCE_DIR}" "${CMAKE_BINARY_DIR}/thirdparty/raylib" EXCLUDE_FROM_ALL)
else()
	message(STATUS "raylib not found in thirdparty, fetching version ${RAYLIB_VERSION}...")
	include(FetchContent)
	FetchContent_Declare(
		raylib
		GIT_REPOSITORY https://github.com/raysan5/raylib.git
		GIT_TAG ${RAYLIB_VERSION}
		GIT_SHALLOW TRUE
		SOURCE_DIR "${RAYLIB_SOURCE_DIR}"
	)
	FetchContent_MakeAvailable(raylib)
endif()

# Dear ImGui's docking branch is vendored inside thirdparty/imgui-docking. It is
# the same library as thirdparty/imgui plus the docking / multi-viewport
# features (ImGuiConfigFlags_DockingEnable, ImGui::DockSpaceOverViewport, ...).
# Only one of the two copies is compiled, because they export the same symbols,
# so the docking branch is used by default. Set IMGUI_USE_DOCKING to OFF to use
# the regular branch in thirdparty/imgui instead; the current main.cpp relies on
# docking-only APIs, so it would then need those calls removed.
#
# The rlImGui backend that renders Dear ImGui with raylib lives in
# thirdparty/rlImGui. All of them are cloned from GitHub during the first CMake
# configure if they aren't there yet.
#
# The versions below are a matched pair: rlImGui's "Raylib_5_5" tag is built
# against raylib 5.5 and Dear ImGui 1.92.7, so the docking tag is pinned to the
# same 1.92.7 release.
set(IMGUI_USE_DOCKING ON CACHE BOOL "Use the docking branch of Dear ImGui")
set(IMGUI_VERSION "v1.92.7" CACHE STRING "Dear ImGui version used when fetching")
set(IMGUI_SOURCE_DIR "${THIRDPARTY_DIR}/imgui")
set(IMGUI_DOCKING_VERSION "v1.92.7-docking" CACHE STRING "Dear ImGui docking branch version used when fetching")
set(IMGUI_DOCKING_SOURCE_DIR "${THIRDPARTY_DIR}/imgui-docking")
set(RLIMGUI_VERSION "Raylib_5_5" CACHE STRING "rlImGui tag used when fetching")
set(RLIMGUI_SOURCE_DIR "${THIRDPARTY_DIR}/rlImGui")

if(IMGUI_USE_DOCKING)
	if(EXISTS "${IMGUI_DOCKING_SOURCE_DIR}/imgui.cpp")
		message(STATUS "Dear ImGui (docking) found at ${IMGUI_DOCKING_SOURCE_DIR}")
	else()
		message(STATUS "Dear ImGui (docking) not found in thirdparty, fetching version ${IMGUI_DOCKING_VERSION}...")
		include(FetchContent)
		FetchContent_Declare(
			imgui-docking
			GIT_REPOSITORY https://github.com/ocornut/imgui.git
			GIT_TAG ${IMGUI_DOCKING_VERSION}
			GIT_SHALLOW TRUE
			SOURCE_DIR "${IMGUI_DOCKING_SOURCE_DIR}"
		)
		FetchContent_MakeAvailable(imgui-docking)
	endif()
	set(IMGUI_ACTIVE_SOURCE_DIR "${IMGUI_DOCKING_SOURCE_DIR}")
else()
	if(EXISTS "${IMGUI_SOURCE_DIR}/imgui.cpp")
		message(STATUS "Dear ImGui found at ${IMGUI_SOURCE_DIR}")
	else()
		message(STATUS "Dear ImGui not found in thirdparty, fetching version ${IMGUI_VERSION}...")
		include(FetchContent)
		FetchContent_Declare(
			imgui
			GIT_REPOSITORY https://github.com/ocornut/imgui.git
			GIT_TAG ${IMGUI_VERSION}
			GIT_SHALLOW TRUE
			SOURCE_DIR "${IMGUI_SOURCE_DIR}"
		)
		FetchContent_MakeAvailable(imgui)
	endif()
	set(IMGUI_ACTIVE_SOURCE_DIR "${IMGUI_SOURCE_DIR}")
endif()

if(EXISTS "${RLIMGUI_SOURCE_DIR}/rlImGui.cpp")
	message(STATUS "rlImGui found at ${RLIMGUI_SOURCE_DIR}")
else()
	message(STATUS "rlImGui not found in thirdparty, fetching version ${RLIMGUI_VERSION}...")
	include(FetchContent)
	FetchContent_Declare(
		rlImGui
		GIT_REPOSITORY https://github.com/raylib-extras/rlImGui.git
		GIT_TAG ${RLIMGUI_VERSION}
		GIT_SHALLOW TRUE
		SOURCE_DIR "${RLIMGUI_SOURCE_DIR}"
	)
	FetchContent_MakeAvailable(rlImGui)
endif()

# Neither upstream project ships a CMakeLists.txt, so build the static
# libraries here. "imgui" is the core library (from whichever branch
# IMGUI_USE_DOCKING selected); "rlImGui" adds the backend that draws ImGui using
# raylib. Link against the "rlImGui" target to use both. imgui_demo.cpp is
# included because main.cpp calls ImGui::ShowDemoWindow().
add_library(imgui STATIC
	"${IMGUI_ACTIVE_SOURCE_DIR}/imgui.cpp"
	"${IMGUI_ACTIVE_SOURCE_DIR}/imgui_demo.cpp"
	"${IMGUI_ACTIVE_SOURCE_DIR}/imgui_draw.cpp"
	"${IMGUI_ACTIVE_SOURCE_DIR}/imgui_tables.cpp"
	"${IMGUI_ACTIVE_SOURCE_DIR}/imgui_widgets.cpp"
)
target_include_directories(imgui PUBLIC "${IMGUI_ACTIVE_SOURCE_DIR}")
set_target_properties(imgui PROPERTIES CXX_STANDARD 17)

add_library(rlImGui STATIC "${RLIMGUI_SOURCE_DIR}/rlImGui.cpp")
target_include_directories(rlImGui PUBLIC "${RLIMGUI_SOURCE_DIR}")
target_link_libraries(rlImGui PUBLIC imgui raylib)
set_target_properties(rlImGui PROPERTIES CXX_STANDARD 17)


# MY_SOURCES is defined to be a list of all the source files for my game 
# DON'T ADD THE SOURCES BY HAND, they are already added with this macro
file(GLOB_RECURSE MY_SOURCES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")

add_executable("${CMAKE_PROJECT_NAME}")

set_property(TARGET "${CMAKE_PROJECT_NAME}" PROPERTY CXX_STANDARD 17)

if(PRODUCTION_BUILD)
	# setup the ASSETS_PATH macro to be in the root folder of your exe
	target_compile_definitions("${CMAKE_PROJECT_NAME}" PUBLIC RESOURCES_PATH="./resources/") 

	# remove the option to debug asserts.
	target_compile_definitions("${CMAKE_PROJECT_NAME}" PUBLIC PRODUCTION_BUILD=1) 

else()
	# This is useful to get an ASSETS_PATH in your IDE during development
	target_compile_definitions("${CMAKE_PROJECT_NAME}" PUBLIC RESOURCES_PATH="${CMAKE_CURRENT_SOURCE_DIR}/resources/")
	target_compile_definitions("${CMAKE_PROJECT_NAME}" PUBLIC PRODUCTION_BUILD=0) 

endif()

target_sources("${CMAKE_PROJECT_NAME}" PRIVATE ${MY_SOURCES} )

# Link raylib (fetched or vendored above)
target_link_libraries("${CMAKE_PROJECT_NAME}" PRIVATE raylib)

# Link Dear ImGui through the rlImGui raylib backend (fetched or vendored above).
# Remove this line if the game doesn't use ImGui; the targets above stay
# available for anything that does.
target_link_libraries("${CMAKE_PROJECT_NAME}" PRIVATE rlImGui)


if(MSVC) # If using the VS compiler...

	target_compile_definitions("${CMAKE_PROJECT_NAME}" PUBLIC _CRT_SECURE_NO_WARNINGS)

	if(PRODUCTION_BUILD)
		#YOU CAN REMOVE THE CONSOLE WITH THIS LINE!
		set_target_properties("${CMAKE_PROJECT_NAME}" PROPERTIES LINK_FLAGS "/SUBSYSTEM:WINDOWS /ENTRY:mainCRTStartup") #no console
	endif()

elseif(MINGW) # Windows / MinGW GCC/Clang
	if(PRODUCTION_BUILD)
		# Remove console on Windows under MinGW
		target_link_options("${CMAKE_PROJECT_NAME}" PRIVATE
			-mwindows
		)
	endif()

elseif(APPLE OR UNIX) # macOS / Linux
	if(PRODUCTION_BUILD)
		target_link_options("${CMAKE_PROJECT_NAME}" PRIVATE -s)
	endif()
endif()

target_include_directories("${CMAKE_PROJECT_NAME}" PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/src/gameLayer")
target_include_directories("${CMAKE_PROJECT_NAME}" PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/src/platform")
