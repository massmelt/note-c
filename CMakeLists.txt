cmake_minimum_required(VERSION 3.13)

if ("${CMAKE_BINARY_DIR}" STREQUAL "${CMAKE_SOURCE_DIR}")
    message(FATAL_ERROR "In-source builds are not allowed.
    Please create a build directory and use `cmake ..` inside it.
    NOTE: cmake will now create CMakeCache.txt and CMakeFiles/*.
          You must delete them, or cmake will refuse to work.")
endif()

project(note_c)

# Automatically ignore CMake build directory.
if(NOT EXISTS ${PROJECT_BINARY_DIR}/.gitignore)
    file(WRITE ${PROJECT_BINARY_DIR}/.gitignore "*")
endif()

option(NOTE_C_BUILD_TESTS "Build tests." OFF)
option(BUILD_SHARED_LIBS "Build shared libraries instead of static." OFF)
option(NOTE_C_COVERAGE "Compile for test NOTE_C_COVERAGE reporting." OFF)
option(NOTE_C_MEM_CHECK "Run tests with Valgrind." OFF)

set(NOTE_C_SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR})
add_library(
    note_c
    ${NOTE_C_SRC_DIR}/n_atof.c
    ${NOTE_C_SRC_DIR}/n_cjson.c
    ${NOTE_C_SRC_DIR}/n_const.c
    ${NOTE_C_SRC_DIR}/n_helpers.c
    ${NOTE_C_SRC_DIR}/n_i2c.c
    ${NOTE_C_SRC_DIR}/n_printf.c
    ${NOTE_C_SRC_DIR}/n_serial.c
    ${NOTE_C_SRC_DIR}/n_ua.c
    ${NOTE_C_SRC_DIR}/n_b64.c
    ${NOTE_C_SRC_DIR}/n_cjson_helpers.c
    ${NOTE_C_SRC_DIR}/n_ftoa.c
    ${NOTE_C_SRC_DIR}/n_hooks.c
    ${NOTE_C_SRC_DIR}/n_md5.c
    ${NOTE_C_SRC_DIR}/n_request.c
    ${NOTE_C_SRC_DIR}/n_str.c
)
target_compile_options(
    note_c
    PRIVATE
        -Wall
        -Wextra
        -Wpedantic
        -Werror
)
target_include_directories(
    note_c
    PUBLIC ${NOTE_C_SRC_DIR}
)

if(NOTE_C_BUILD_TESTS)
    # Including CTest here rather than in test/CMakeLists.txt allows us to run
    # ctest from the root build directory (e.g. build/ instead of build/test/).
    # We also need to set MEMORYCHECK_COMMAND_OPTIONS before including this.
    # See: https://stackoverflow.com/a/60741757
    if(NOTE_C_MEM_CHECK)
        # Go ahead and make sure we can find valgrind while we're here.
        find_program(VALGRIND valgrind REQUIRED)
        message(STATUS "Found valgrind: ${VALGRIND}")
        set(MEMORYCHECK_COMMAND_OPTIONS "--leak-check=full --error-exitcode=1")
    endif(NOTE_C_MEM_CHECK)
    include(CTest)

    target_compile_definitions(
        note_c
        PUBLIC NOTE_C_TEST
    )
    target_include_directories(
        note_c
        PRIVATE ${CMAKE_CURRENT_LIST_DIR}/test/include
    )

    # If we don't weaken the functions we're mocking in the tests, the linker
    # will complain about multiple function definitions: the mocked one and the
    # "real" one from note-c. Weakening the real function causes the mock
    # function, if defined, to override the real one. If no mock is defined, the
    # real one will be used. So, every time a developer needs to mock a function
    # in a test, they need to make sure it's included in the MOCKED_FNS list
    # below.
    set(
        MOCKED_FNS
        "NoteReset;
         NoteJSONTransaction;
         NoteTransaction;
         NoteGetMs;
         NoteRequestResponse;
         NoteMalloc;
         NoteI2CTransmit;
         NoteI2CReceive;
         NoteLockI2C;
         NoteUnlockI2C;
         NoteI2CReset;
         NoteSerialAvailable;
         NoteSerialTransmit;
         NoteSerialReceive;
         NoteSerialReset;
         NoteIsDebugOutputActive;
         NoteDebug;
         NotePrint;
         NoteNewCommand;
         NoteRequest;
         JCreateObject;
         NoteTransactionStart;
         NoteUserAgent;
         serialNoteReset;
         serialNoteTransaction;
         i2cNoteReset;
         i2cNoteTransaction;
         JCreateStringValue;
         NoteSetEnvDefault;
         NoteSleep;
         NotePayloadRetrieveAfterSleep;
         NoteTimeValidST;
         NoteTimeST;
         NoteRegion"
    )
    foreach(MOCKED_FN ${MOCKED_FNS})
        string(APPEND OBJCOPY_WEAKEN "-W ${MOCKED_FN} ")
    endforeach()
    separate_arguments(OBJCOPY_WEAKEN_LIST NATIVE_COMMAND "${OBJCOPY_WEAKEN}")
    add_custom_command(TARGET note_c POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} ${OBJCOPY_WEAKEN_LIST}
        $<TARGET_FILE:note_c>
        COMMENT "Weakening mocked functions."
    )

    add_subdirectory(test)
endif(NOTE_C_BUILD_TESTS)
