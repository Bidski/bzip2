CMAKE_MINIMUM_REQUIRED(VERSION 3.0)
PROJECT(BZip2)

# Set on to prefer .a files over .so files for linking
OPTION(STATIC_LIBRARIES "Prefer using static .a files rather than .so files. NOTE: This must be enabled from the start!" OFF)

IF(STATIC_LIBRARIES)
    SET(CMAKE_FIND_LIBRARY_SUFFIXES
        ${CMAKE_STATIC_LIBRARY_SUFFIX}
        ${CMAKE_SHARED_LIBRARY_SUFFIX})
ENDIF()

OPTION(BUILD_BINARIES "Build executable binaries" ON)
OPTION(BUILD_DOCS "Build docs" ON)
OPTION(RUN_TESTS "Run tests" ON)

# Set some install directories
SET(INSTALL_PKGCONFIG_DIR "${CMAKE_INSTALL_PREFIX}/share/pkgconfig" CACHE PATH "Installation directory for pkgconfig (.pc) files")
SET(INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib" CACHE PATH "Installation directory for libraries")
SET(INSTALL_INC_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE PATH "Installation directory for headers")
SET(INSTALL_MAN_DIR "${CMAKE_INSTALL_PREFIX}/share/man" CACHE PATH "Installation directory for manual pages")


IF (RUN_TESTS AND NOT BUILD_BINARIES)
    SET(BUILD_BINARIES ON)
ENDIF()


SET(SRCS ${CMAKE_SOURCE_DIR}/blocksort.c
         ${CMAKE_SOURCE_DIR}/huffman.c
         ${CMAKE_SOURCE_DIR}/crctable.c
         ${CMAKE_SOURCE_DIR}/randtable.c
         ${CMAKE_SOURCE_DIR}/compress.c
         ${CMAKE_SOURCE_DIR}/decompress.c
         ${CMAKE_SOURCE_DIR}/bzlib.c)

ADD_LIBRARY(bz2_shared SHARED ${SRCS})
ADD_LIBRARY(bz2_static STATIC ${SRCS})

# Rename shared library
SET_TARGET_PROPERTIES(bz2_shared PROPERTIES OUTPUT_NAME "bz2")
SET_TARGET_PROPERTIES(bz2_shared PROPERTIES SUFFIX ".so.1.0.6")
SET_TARGET_PROPERTIES(bz2_static PROPERTIES OUTPUT_NAME "bz2")

# Copy header file to binary directory
ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzlib.h"
                   COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_SOURCE_DIR}/bzlib.h" "${CMAKE_BINARY_DIR}/bzlib.h"
)
ADD_CUSTOM_TARGET(copy_header ALL DEPENDS "${CMAKE_BINARY_DIR}/bzlib.h")

# Create shared library symlinks
ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/libbz2.so.1.0"
                   COMMAND ${CMAKE_COMMAND} -E create_symlink "libbz2.so.1.0.6" "libbz2.so.1.0"
                   DEPENDS bz2_shared
                   WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
)

ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/libbz2.so.1"
                   COMMAND ${CMAKE_COMMAND} -E create_symlink "libbz2.so.1.0" "libbz2.so.1"
                   DEPENDS "${CMAKE_BINARY_DIR}/libbz2.so.1.0"
                   WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
)

ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/libbz2.so"
                   COMMAND ${CMAKE_COMMAND} -E create_symlink "libbz2.so.1" "libbz2.so"
                   DEPENDS "${CMAKE_BINARY_DIR}/libbz2.so.1"
                   WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
)
ADD_CUSTOM_TARGET(shared_links ALL DEPENDS "${CMAKE_BINARY_DIR}/libbz2.so"
                                           "${CMAKE_BINARY_DIR}/libbz2.so.1"
                                           "${CMAKE_BINARY_DIR}/libbz2.so.1.0"
                                           bz2_shared
)

IF (BUILD_BINARIES)
    ADD_EXECUTABLE(bzip2 "${CMAKE_SOURCE_DIR}/bzip2.c")
    TARGET_LINK_LIBRARIES(bzip2 bz2)

    ADD_EXECUTABLE(bunzip2 "${CMAKE_SOURCE_DIR}/bzip2.c")
    TARGET_LINK_LIBRARIES(bunzip2 bz2)

    ADD_EXECUTABLE(bzcat "${CMAKE_SOURCE_DIR}/bzip2.c")
    TARGET_LINK_LIBRARIES(bzcat bz2)

    ADD_EXECUTABLE(bzip2recover "${CMAKE_SOURCE_DIR}/bzip2recover.c")

    # Copy scripts into binary directory
    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzgrep"
                       COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_SOURCE_DIR}/bzgrep" "${CMAKE_BINARY_DIR}/bzgrep"
    )

    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzmore"
                       COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_SOURCE_DIR}/bzmore" "${CMAKE_BINARY_DIR}/bzmore"
    )

    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzdiff"
                       COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_SOURCE_DIR}/bzdiff" "${CMAKE_BINARY_DIR}/bzdiff"
    )
    ADD_CUSTOM_TARGET(copy_scripts ALL DEPENDS "${CMAKE_BINARY_DIR}/bzgrep"
                                               "${CMAKE_BINARY_DIR}/bzmore"
                                               "${CMAKE_BINARY_DIR}/bzdiff"
    )

    # Create symlinks for scripts
    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzegrep"
                       COMMAND ${CMAKE_COMMAND} -E create_symlink "bzgrep" "bzegrep"
                       DEPENDS "${CMAKE_BINARY_DIR}/bzgrep"
                       WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
    )

    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzfgrep"
                       COMMAND ${CMAKE_COMMAND} -E create_symlink "bzgrep" "bzfgrep"
                       DEPENDS "${CMAKE_BINARY_DIR}/bzgrep"
                       WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
    )

    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzless"
                       COMMAND ${CMAKE_COMMAND} -E create_symlink "bzmore" "bzless"
                       DEPENDS "${CMAKE_BINARY_DIR}/bzmore"
                       WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
    )

    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzcmp"
                       COMMAND ${CMAKE_COMMAND} -E create_symlink "bzdiff" "bzcmp"
                       DEPENDS "${CMAKE_BINARY_DIR}/bzdiff"
                       WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
    )
    ADD_CUSTOM_TARGET(script_links ALL DEPENDS "${CMAKE_BINARY_DIR}/bzegrep"
                                               "${CMAKE_BINARY_DIR}/bzfgrep"
                                               "${CMAKE_BINARY_DIR}/bzless"
                                               "${CMAKE_BINARY_DIR}/bzcmp"
    )
ENDIF()

IF (RUN_TESTS)
    ENABLE_TESTING()

    FOREACH(iter 1 2 3)
        # Comperssion tests
        ADD_TEST(NAME Compress${iter}a
                 COMMAND "${CMAKE_COMMAND}" -E copy "${CMAKE_SOURCE_DIR}/sample${iter}.ref" "${CMAKE_BINARY_DIR}/sample${iter}.ref")
        ADD_TEST(NAME Compress${iter}b
                 COMMAND "${CMAKE_COMMAND}" -E copy "${CMAKE_SOURCE_DIR}/sample${iter}.bz2" "${CMAKE_BINARY_DIR}/sample${iter}.bz2")
        ADD_TEST(NAME Compress${iter}c
                 COMMAND "${CMAKE_BINARY_DIR}/bzip2" -k -f -${iter} "${CMAKE_BINARY_DIR}/sample${iter}.ref")
        ADD_TEST(NAME Compress${iter}d
                 COMMAND ${CMAKE_COMMAND} -E compare_files "${CMAKE_BINARY_DIR}/sample${iter}.bz2" "${CMAKE_BINARY_DIR}/sample${iter}.ref.bz2")
        SET_TESTS_PROPERTIES(Compress${iter}b PROPERTIES DEPENDS Compress${iter}a)
        SET_TESTS_PROPERTIES(Compress${iter}c PROPERTIES DEPENDS Compress${iter}b)
        SET_TESTS_PROPERTIES(Compress${iter}d PROPERTIES DEPENDS Compress${iter}c)

        # Decomperssion tests
        IF (${iter} EQUAL 3)
            ADD_TEST(NAME Decompress${iter}a
                     COMMAND "${CMAKE_BINARY_DIR}/bzip2" -k -f -ds "${CMAKE_BINARY_DIR}/sample${iter}.bz2")
        ELSE()
            ADD_TEST(NAME Decompress${iter}a
                     COMMAND "${CMAKE_BINARY_DIR}/bzip2" -k -f -d "${CMAKE_BINARY_DIR}/sample${iter}.bz2")
        ENDIF()

        ADD_TEST(NAME Decompress${iter}b
                 COMMAND ${CMAKE_COMMAND} -E compare_files "${CMAKE_BINARY_DIR}/sample${iter}.ref" "${CMAKE_BINARY_DIR}/sample${iter}")
        SET_TESTS_PROPERTIES(Decompress${iter}a PROPERTIES DEPENDS Compress${iter}b)
        SET_TESTS_PROPERTIES(Decompress${iter}b PROPERTIES DEPENDS Decompress${iter}a)
    ENDFOREACH()
ENDIF()

IF (BUILD_DOCS)
    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzip2.1"
                       COMMAND "${CMAKE_COMMAND}" -E copy "${CMAKE_SOURCE_DIR}/bzip2.1" "${CMAKE_BINARY_DIR}/bzip2.1"
    )

    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzgrep.1"
                       COMMAND "${CMAKE_COMMAND}" -E copy "${CMAKE_SOURCE_DIR}/bzgrep.1" "${CMAKE_BINARY_DIR}/bzgrep.1"
    )

    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzmore.1"
                       COMMAND "${CMAKE_COMMAND}" -E copy "${CMAKE_SOURCE_DIR}/bzmore.1" "${CMAKE_BINARY_DIR}/bzmore.1"
    )

    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzdiff.1"
                       COMMAND "${CMAKE_COMMAND}" -E copy "${CMAKE_SOURCE_DIR}/bzdiff.1" "${CMAKE_BINARY_DIR}/bzdiff.1"
    )

    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzegrep.1"
                       COMMAND echo ".so man1/bzgrep.1" > "${CMAKE_BINARY_DIR}/bzegrep.1"
    )

    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzfgrep.1"
                       COMMAND echo ".so man1/bzgrep.1" > "${CMAKE_BINARY_DIR}/bzfgrep.1"
    )

    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzless.1"
                       COMMAND echo ".so man1/bzmore.1" > "${CMAKE_BINARY_DIR}/bzless.1"
    )

    ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_BINARY_DIR}/bzcmp.1"
                       COMMAND echo ".so man1/bzdiff.1" > "${CMAKE_BINARY_DIR}/bzcmp.1"
    )
    ADD_CUSTOM_TARGET(man_files ALL DEPENDS "${CMAKE_BINARY_DIR}/bzip2.1"
                                            "${CMAKE_BINARY_DIR}/bzgrep.1"
                                            "${CMAKE_BINARY_DIR}/bzmore.1"
                                            "${CMAKE_BINARY_DIR}/bzdiff.1"
                                            "${CMAKE_BINARY_DIR}/bzegrep.1"
                                            "${CMAKE_BINARY_DIR}/bzfgrep.1"
                                            "${CMAKE_BINARY_DIR}/bzless.1"
                                            "${CMAKE_BINARY_DIR}/bzcmp.1"
    )
ENDIF()

INSTALL(TARGETS bz2_shared bz2_static
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
)

INSTALL(FILES "${CMAKE_BINARY_DIR}/libbz2.so"
              "${CMAKE_BINARY_DIR}/libbz2.so.1"
              "${CMAKE_BINARY_DIR}/libbz2.so.1.0"
              "${CMAKE_BINARY_DIR}/libbz2.so.1.0.6"
        DESTINATION lib
)

IF (BUILD_BINARIES)
    INSTALL(TARGETS bzip2 bzip2recover bunzip2 bzcat
            RUNTIME DESTINATION bin
    )

    INSTALL(FILES "${CMAKE_BINARY_DIR}/bzgrep"
                  "${CMAKE_BINARY_DIR}/bzmore"
                  "${CMAKE_BINARY_DIR}/bzdiff"
                  "${CMAKE_BINARY_DIR}/bzegrep"
                  "${CMAKE_BINARY_DIR}/bzfgrep"
                  "${CMAKE_BINARY_DIR}/bzless"
                  "${CMAKE_BINARY_DIR}/bzcmp"
            DESTINATION bin
    )
ENDIF()

INSTALL(FILES "${CMAKE_BINARY_DIR}/bzlib.h"
        DESTINATION include
)

IF (BUILD_DOCS)
    INSTALL(FILES "${CMAKE_BINARY_DIR}/bzip2.1"
                  "${CMAKE_BINARY_DIR}/bzgrep.1"
                  "${CMAKE_BINARY_DIR}/bzmore.1"
                  "${CMAKE_BINARY_DIR}/bzdiff.1"
                  "${CMAKE_BINARY_DIR}/bzegrep.1"
                  "${CMAKE_BINARY_DIR}/bzfgrep.1"
                  "${CMAKE_BINARY_DIR}/bzless.1"
                  "${CMAKE_BINARY_DIR}/bzcmp.1"
            DESTINATION man/man1
    )
ENDIF()

CONFIGURE_FILE("${CMAKE_SOURCE_DIR}/bzip2.pc.cmakein" "${CMAKE_BINARY_DIR}/bzip2.pc" @ONLY)
INSTALL(FILES "${CMAKE_BINARY_DIR}/bzip2.pc" DESTINATION "${INSTALL_PKGCONFIG_DIR}")

