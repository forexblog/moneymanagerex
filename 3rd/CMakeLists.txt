IF(NOT InMMEX)
    MESSAGE(FATAL_ERROR "Use the top-level CMake script!")
ENDIF(NOT InMMEX)

# disable all warnings while compile sources in 3rd
IF(MSVC)
    SET_DIRECTORY_PROPERTIES(PROPERTIES COMPILE_OPTIONS /W0)
ELSE()
    SET_DIRECTORY_PROPERTIES(PROPERTIES COMPILE_OPTIONS -w)
ENDIF()

ADD_LIBRARY(RapidJSON INTERFACE)
TARGET_INCLUDE_DIRECTORIES(RapidJSON INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/rapidjson/include>)

ADD_LIBRARY(csv_parser INTERFACE)
TARGET_INCLUDE_DIRECTORIES(csv_parser INTERFACE csv-parser/include)

ADD_LIBRARY(LuaGlue INTERFACE)
TARGET_INCLUDE_DIRECTORIES(LuaGlue INTERFACE LuaGlue/include)

FIND_PACKAGE(Lua QUIET)
IF(LUA_FOUND)
    MESSAGE(STATUS "Found Lua: ${LUA_LIBRARIES} (found version \"${LUA_VERSION_STRING}\")") 
    ADD_LIBRARY(Lua INTERFACE)
    TARGET_INCLUDE_DIRECTORIES(Lua INTERFACE ${LUA_INCLUDE_DIR})
    TARGET_LINK_LIBRARIES(Lua INTERFACE ${LUA_LIBRARIES})
ELSE()
    ADD_LIBRARY(Lua STATIC EXCLUDE_FROM_ALL
        lua/lapi.c
        lua/lauxlib.c
        lua/lbaselib.c
        lua/lbitlib.c
        lua/lcode.c
        lua/lcorolib.c
        lua/lctype.c
        lua/ldblib.c
        lua/ldebug.c
        lua/ldo.c
        lua/ldump.c
        lua/lfunc.c
        lua/lgc.c
        lua/linit.c
        lua/liolib.c
        lua/llex.c
        lua/lmathlib.c
        lua/lmem.c
        lua/loadlib.c
        lua/lobject.c
        lua/lopcodes.c
        lua/loslib.c
        lua/lparser.c
        lua/lstate.c
        lua/lstring.c
        lua/lstrlib.c
        lua/ltable.c
        lua/ltablib.c
        lua/ltm.c
        lua/lua.c
        lua/lundump.c
        lua/lutf8lib.c
        lua/lvm.c
        lua/lzio.c)
    #add C++ wrapper for Lua include files
    CONFIGURE_FILE(${PROJECT_SOURCE_DIR}/resources/lua.hpp lua.hpp COPYONLY)
    TARGET_INCLUDE_DIRECTORIES(Lua INTERFACE lua ${CMAKE_CURRENT_BINARY_DIR})
    IF(NOT WIN32)
        TARGET_COMPILE_DEFINITIONS(Lua PRIVATE
            LUA_USE_POSIX
            LUA_USE_DLOPEN
            LUA_USE_STRTODHEX
            LUA_USE_AFORMAT
            LUA_USE_LONGLONG)
    ENDIF()

    IF(LINUX)
        INCLUDE(CheckLibraryExists)
        CHECK_LIBRARY_EXISTS(${CMAKE_DL_LIBS} dlopen "" HAVE_DLOPEN)
        IF(NOT HAVE_DLOPEN)
            MESSAGE(SEND_ERROR "Could not find required dl library.")
        ENDIF()
        TARGET_LINK_LIBRARIES(Lua PRIVATE ${CMAKE_DL_LIBS})
    ENDIF()
ENDIF()
TARGET_COMPILE_DEFINITIONS(Lua INTERFACE LUA_COMPAT_5_2)

ADD_LIBRARY(HTML-template STATIC EXCLUDE_FROM_ALL cgitemplate/html_template.cpp)
TARGET_INCLUDE_DIRECTORIES(HTML-template INTERFACE cgitemplate)

ADD_LIBRARY(Mongoose STATIC EXCLUDE_FROM_ALL mongoose/mongoose.c)
TARGET_INCLUDE_DIRECTORIES(Mongoose INTERFACE .)
TARGET_COMPILE_DEFINITIONS(Mongoose
    PRIVATE
        # MG_ENABLE_SSL
        MG_ENABLE_HTTP_WEBSOCKET=0
        MG_DISABLE_HTTP_DIGEST_AUTH
        CS_DISABLE_SHA1
        CS_DISABLE_MD5)

FIND_PACKAGE(PkgConfig)
IF(PkgConfig_FOUND)
    PKG_SEARCH_MODULE(wxSQLite3 wxsqlite3-3.0 wxsqlite3 wxsqlite>=3)
ENDIF()
IF(wxSQLite3_FOUND)
    ADD_LIBRARY(wxSQLite3 INTERFACE)
    TARGET_INCLUDE_DIRECTORIES(wxSQLite3 INTERFACE ${wxSQLite3_INCLUDE_DIRS})
    TARGET_LINK_LIBRARIES(wxSQLite3 INTERFACE ${wxSQLite3_LIBRARIES} wxWidgets)
    TARGET_COMPILE_OPTIONS(wxSQLite3 INTERFACE ${wxSQLite3_CFLAGS_OTHER})
ELSE()
    ADD_LIBRARY(wxSQLite3 STATIC EXCLUDE_FROM_ALL
        wxsqlite3/src/wxsqlite3.cpp
        wxsqlite3/sqlite3/secure/src/sqlite3secure.c)
    TARGET_INCLUDE_DIRECTORIES(wxSQLite3
        PUBLIC wxsqlite3/include wxsqlite3/sqlite3/secure/src)
    TARGET_LINK_LIBRARIES(wxSQLite3 PUBLIC wxWidgets)

    IF(LINUX)
        INCLUDE(CheckLibraryExists)
        CHECK_LIBRARY_EXISTS(${CMAKE_DL_LIBS} dlopen "" HAVE_DLOPEN)
        IF(NOT HAVE_DLOPEN)
            MESSAGE(SEND_ERROR "Could not find required dl library.")
        ENDIF()
        TARGET_LINK_LIBRARIES(wxSQLite3 PRIVATE ${CMAKE_DL_LIBS})
    ENDIF()

    TARGET_COMPILE_DEFINITIONS(wxSQLite3
        PRIVATE
            NOPCH
            SQLITE_CORE
            SQLITE_ENABLE_FTS3
            SQLITE_ENABLE_FTS3_PARENTHESIS
            SQLITE_HAS_CODEC
            SQLITE_ENABLE_EXTFUNC
            SQLITE_ENABLE_COLUMN_METADATA
            SQLITE_ENABLE_JSON1
            HAVE_ACOSH
            HAVE_ASINH
            HAVE_ATANH
            HAVE_ISBLANK
        PUBLIC
            WXSQLITE3_HAVE_CODEC
            WXSQLITE3_HAVE_METADATA
            WXSQLITE3_USER_AUTHENTICATION)

    IF(CMAKE_VERSION VERSION_LESS 3.1)
        INCLUDE(CheckCXXCompilerFlag)
        CHECK_CXX_COMPILER_FLAG("-std=gnu++11" COMPILER_SUPPORTS_GXX11)
        CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
        CHECK_CXX_COMPILER_FLAG("-std=gnu++0x" COMPILER_SUPPORTS_GXX0X)
        CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
        IF(COMPILER_SUPPORTS_GXX11)
            TARGET_COMPILE_OPTIONS(wxSQLite3 PUBLIC -std=gnu++11)
        ELSEIF(COMPILER_SUPPORTS_CXX11)
            TARGET_COMPILE_OPTIONS(wxSQLite3 PUBLIC -std=c++11)
        ELSEIF(COMPILER_SUPPORTS_GXX0X)
            TARGET_COMPILE_OPTIONS(wxSQLite3 PUBLIC -std=gnu++0x)
        ELSEIF(COMPILER_SUPPORTS_CXX0X)
            TARGET_COMPILE_OPTIONS(wxSQLite3 PUBLIC -std=c++0x)
        ELSE()
            MESSAGE(SEND_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support.")
        ENDIF()
    ELSEIF(CMAKE_VERSION VERSION_LESS 3.8)
        TARGET_COMPILE_FEATURES(wxSQLite3 PUBLIC cxx_range_for)
    ELSE()
        TARGET_COMPILE_FEATURES(wxSQLite3 PUBLIC cxx_std_11)
    ENDIF()        
ENDIF()


IF(MMEX_BUILD_TESTS)
    INCLUDE(ExternalProject)
    ADD_LIBRARY(CPPUnitLib STATIC IMPORTED GLOBAL)
    
    IF(MSVC)
        # hack to build under latest VS version
        FILE(READ cppunit/Build2010.sln contents)
        SET(BuildCppunitMSI B032B761-88DF-4170-A1D0-F89E806FFCAA)
        STRING(REGEX REPLACE "\nProject[^\n]*${BuildCppunitMSI}[^\n]*\nEndProject\n" "\n" contents "${contents}")
        STRING(REGEX REPLACE "\n[^\n]*${BuildCppunitMSI}[^\n]*\n" "\n" contents "${contents}")
        FILE(WRITE cppunit/Build2010.sln "${contents}")
        UNSET(contents)
        UNSET(BuildCppunitMSI)
        
        FIND_PROGRAM(devenv devenv)
        IF (NOT devenv)
            MESSAGE(SEND_ERROR "devenv tool is required for MMEX_BUILD_TESTS but not found.")
        ENDIF()

        ExternalProject_ADD(CPPUnit
            SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/cppunit"
            CONFIGURE_COMMAND ${devenv} /upgrade Build2010.sln
            BUILD_COMMAND ${devenv} Build2010.sln /Build "$<CONFIG>|Win32" /Project cppunit
            BUILD_IN_SOURCE 1)
        SET(cppunit_lib "${CMAKE_CURRENT_SOURCE_DIR}/cppunit/lib/${CMAKE_STATIC_LIBRARY_PREFIX}cppunit")
        SET_TARGET_PROPERTIES(CPPUnitLib PROPERTIES
            IMPORTED_LOCATION_RELEASE "${cppunit_lib}${CMAKE_STATIC_LIBRARY_SUFFIX}"
            IMPORTED_LOCATION_DEBUG "${cppunit_lib}d${CMAKE_STATIC_LIBRARY_SUFFIX}")    
        UNSET(cppunit_lib)    
    ELSE(MSVC)
        FIND_PROGRAM(aclocal aclocal)
        FIND_PROGRAM(libtoolize NAMES libtoolize glibtoolize)
        FIND_PROGRAM(autoheader autoheader)
        FIND_PROGRAM(automake automake)
        FIND_PROGRAM(autoconf autoconf)
        FIND_PROGRAM(make make)
        IF (NOT (aclocal AND libtoolize AND autoheader
            AND automake AND autoconf AND make))
            MESSAGE(SEND_ERROR "libtool and automake tools are required for MMEX_BUILD_TESTS but not found.")
        ENDIF()

        ExternalProject_ADD(CPPUnit
            SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/cppunit"
            CONFIGURE_COMMAND ./autogen.sh && ./configure -q --disable-shared --disable-doxygen --enable-silent-rules "--prefix=${PROJECT_BINARY_DIR}/cppunit"
            INSTALL_COMMAND ${make} CXXFLAGS='-Wno-deprecated-declarations' install-exec
            BUILD_IN_SOURCE 1)
        SET_TARGET_PROPERTIES(CPPUnitLib PROPERTIES
            IMPORTED_LOCATION "${PROJECT_BINARY_DIR}/cppunit/lib/${CMAKE_STATIC_LIBRARY_PREFIX}cppunit${CMAKE_STATIC_LIBRARY_SUFFIX}")
    ENDIF(MSVC)
    ADD_DEPENDENCIES(CPPUnitLib CPPUnit)
    SET_TARGET_PROPERTIES(CPPUnitLib PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_CURRENT_SOURCE_DIR}/cppunit/include"
        IMPORTED_LINK_INTERFACE_LANGUAGES CXX)

        # hide warnings from cppunit includes
    IF(CMAKE_COMPILER_IS_GNUCXX)
        SET_TARGET_PROPERTIES(CPPUnitLib PROPERTIES
            INTERFACE_COMPILE_OPTIONS -Wno-deprecated-declarations)
    ENDIF()
    
ENDIF(MMEX_BUILD_TESTS)
