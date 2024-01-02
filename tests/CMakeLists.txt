# 日志测试
add_executable(AsyncLoggingTest ${LOG_SRC} ${CMAKE_SOURCE_DIR}/tests/AsyncLoggingTest.cpp)
target_link_libraries(AsyncLoggingTest pthread)
