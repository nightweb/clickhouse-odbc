add_subdirectory(escaping)
if (WIN32)
    add_subdirectory(win)
    #TODO: set(WIN_SOURCES "win/export.def")
endif()

function(makelib libname UNICODE)

#option (UNICODE "Enable unicode" 0)
if (UNICODE)
    set(UNICODE 1)
    if (ODBC_UNIXODBC)
        set(SQL_WCHART_CONVERT 1)
    endif ()
    set(_config_add w)
else ()
    set(UNICODE 0)
endif ()

# Hell matrix
#               TCHAR       sizeof()    note
# not unicode   char        1
# unicode:
# win           wchar_t     2
# iodbc         wchar_t     4
# unixodbc      wchar_t     4           true == SQL_WCHART_CONVERT
# unixodbc      char16_t    2           false == SQL_WCHART_CONVERT

if (UNICODE)
    if (ODBC_UNIXODBC AND NOT SQL_WCHART_CONVERT)
        # char16
    else()
        set(ODBC_WCHAR 1)
    endif()
endif()

message (STATUS "${libname}: UNICODE=${UNICODE} SQL_WCHART_CONVERT=${SQL_WCHART_CONVERT} ODBC_WCHAR=${ODBC_WCHAR}")

configure_file (${CMAKE_CURRENT_SOURCE_DIR}/config_cmake.h.in ${CMAKE_CURRENT_BINARY_DIR}/config_cmake${_config_add}.h)

add_library(${libname} SHARED
    attr.cpp
    config.cpp
    connection.cpp
    diagnostics.cpp
    environment.cpp
    handles.cpp
    info.cpp
    odbc.cpp
    read_helpers.cpp
    result_set.cpp
    statement.cpp
    type_parser.cpp
    log.cpp
    ${WIN_SOURCES}
)


if (UNICODE)
    target_compile_definitions(${libname} PUBLIC -DUNICODE=1 -D_UNICODE=1)
    # Mode for unixodbc for working with wchar as in iodbc
    if (SQL_WCHART_CONVERT)
        target_compile_definitions(${libname} PUBLIC -DSQL_WCHART_CONVERT=1)
    endif ()
endif ()

target_compile_definitions (${libname} PRIVATE "-DCMAKE_BUILD=1")

target_include_directories (${libname} PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

if (NOT USE_INTERNAL_POCO_LIBRARY)
    target_include_directories (${libname} PRIVATE ${Poco_INCLUDE_DIRS})
endif()

if (NOT WIN32)
    target_include_directories (${libname} PRIVATE ${ODBC_INCLUDE_DIRECTORIES})
endif()

target_link_libraries(${libname} clickhouse-odbc-escaping)

if(USE_SSL)
    target_link_libraries(${libname} ${Poco_NetSSL_LIBRARY} ${Poco_Crypto_LIBRARY} ${Poco_Util_LIBRARY})
    target_include_directories (${libname} PRIVATE ${OPENSSL_INCLUDE_DIR})
endif()

target_link_libraries(${libname}
    ${Poco_Net_LIBRARY}
    ${Poco_Foundation_LIBRARY}
    ${ODBC_LIBRARIES}
)

if(NOT APPLE AND NOT WIN32)
    target_link_libraries(${libname} "-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/linker_script")
endif()

if (WIN32)
  target_link_libraries(${libname} clickhouse-odbc-win ws2_32 Iphlpapi legacy_stdio_definitions)
endif()

if (NOT WIN32)
    install(TARGETS ${libname} LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}/odbc)
endif()

endfunction()

makelib(clickhouse-odbc 0)
makelib(clickhouse-odbcw 1)

set_target_properties(clickhouse-odbc PROPERTIES OUTPUT_NAME "clickhouseodbc")
set_target_properties(clickhouse-odbcw PROPERTIES OUTPUT_NAME "clickhouseodbcw")

if (NOT WIN32)
    install(FILES unixodbc.ini DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/clickhouse-odbc)
    install(FILES odbc.ini DESTINATION ${CMAKE_INSTALL_DOCDIR}/examples)
endif()

if(CLICKHOUSE_ODBC_TEST)
    add_subdirectory(ut)
endif()
