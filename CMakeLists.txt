cmake_minimum_required(VERSION 3.5)
project(serial)

set(DEFAULT_USE_AMENT_CMAKE OFF)
if(DEFINED ENV{AMENT_PREFIX_PATH} AND NOT DEFINED ENV{CATKIN_DEVEL_PREFIX})
  set(DEFAULT_USE_AMENT_CMAKE ON)
endif()
option(USE_AMENT_CMAKE "Use ament_cmake to configure the library as an ament package" ${DEFAULT_USE_AMENT_CMAKE})

if(APPLE)
  find_library(IOKIT_LIBRARY IOKit)
  find_library(FOUNDATION_LIBRARY Foundation)
endif()

set(serial_SRCS
    src/serial.cc
    include/serial/serial.h
    include/serial/v8stdint.h
)
if(APPLE)
    list(APPEND serial_SRCS src/impl/unix.cc)
    list(APPEND serial_SRCS src/impl/list_ports/list_ports_osx.cc)
elseif(UNIX)
    list(APPEND serial_SRCS src/impl/unix.cc)
    list(APPEND serial_SRCS src/impl/list_ports/list_ports_linux.cc)
else()
    list(APPEND serial_SRCS src/impl/win.cc)
    list(APPEND serial_SRCS src/impl/list_ports/list_ports_win.cc)
endif()

add_library(${PROJECT_NAME} ${serial_SRCS})
target_include_directories(${PROJECT_NAME}
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

if(APPLE)
    target_link_libraries(${PROJECT_NAME} PUBLIC ${FOUNDATION_LIBRARY} ${IOKIT_LIBRARY})
elseif(UNIX)
    target_link_libraries(${PROJECT_NAME} PUBLIC rt pthread)
else()
    target_link_libraries(${PROJECT_NAME} PUBLIC setupapi)
endif()

add_executable(serial_example examples/serial_example.cc)
add_dependencies(serial_example ${PROJECT_NAME})
target_link_libraries(serial_example ${PROJECT_NAME})

if(USE_AMENT_CMAKE)
  find_package(ament_cmake REQUIRED)

  install(TARGETS ${PROJECT_NAME}
    EXPORT export_${PROJECT_NAME}
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES DESTINATION include
  )

  install(EXPORT export_${PROJECT_NAME}
    DESTINATION share/${PROJECT_NAME}
  )

  install(FILES include/serial/serial.h include/serial/v8stdint.h
    DESTINATION include/serial
  )

  ament_export_targets(export_${PROJECT_NAME} HAS_LIBRARY_TARGET)
  ament_export_libraries(${PROJECT_NAME})
  ament_export_include_directories(include)

  if(BUILD_TESTING)
    find_package(ament_cmake_gtest REQUIRED)
    add_subdirectory(tests)
  endif()

  ament_package()
else()
  find_package(catkin REQUIRED)

  if(UNIX AND NOT APPLE)
      set(rt_LIBRARIES rt)
      set(pthread_LIBRARIES pthread)
      catkin_package(
          LIBRARIES ${PROJECT_NAME}
          INCLUDE_DIRS include
          DEPENDS rt pthread
      )
  else()
      catkin_package(
          LIBRARIES ${PROJECT_NAME}
          INCLUDE_DIRS include
      )
  endif()

  include_directories(${catkin_INCLUDE_DIRS})

  install(TARGETS ${PROJECT_NAME}
      ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
      LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  )

  install(FILES include/serial/serial.h include/serial/v8stdint.h
    DESTINATION ${CATKIN_GLOBAL_INCLUDE_DESTINATION}/serial
  )

  if(CATKIN_ENABLE_TESTING)
      add_subdirectory(tests)
  endif()
endif()
