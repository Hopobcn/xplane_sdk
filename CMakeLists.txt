# Set up the imported targets for X-Plane plugin libraries:

if(APPLE)
    set(XPLANE_LIBRARY_PATH "${CMAKE_CURRENT_SOURCE_DIR}/sdk/Libraries/Mac/")
elseif(WIN32)
    set(XPLANE_LIBRARY_PATH "${CMAKE_CURRENT_SOURCE_DIR}/sdk/Libraries/Win/")
endif()

set(XPLANE_INCLUDES_PATH "${CMAKE_CURRENT_SOURCE_DIR}/sdk/CHeaders/" PARENT_SCOPE)

if(APPLE OR WIN32)

    if(APPLE)
        find_library(XPLM_LIBRARY XPLM "${XPLANE_LIBRARY_PATH}")
        find_library(XPWIDGETS_LIBRARY XPWidgets "${XPLANE_LIBRARY_PATH}")
        add_library(xplm STATIC IMPORTED GLOBAL)
        add_library(xpwidgets STATIC IMPORTED GLOBAL)
        set_property(TARGET xplm PROPERTY IMPORTED_LOCATION "${XPLM_LIBRARY}")
        set_property(TARGET xpwidgets PROPERTY IMPORTED_LOCATION "${XPWIDGETS_LIBRARY}")
    elseif(WIN32)
        find_library(XPLM_LIBRARY XPLM_64 "${XPLANE_LIBRARY_PATH}")
        find_library(XPWIDGETS_LIBRARY XPWidgets_64 "${XPLANE_LIBRARY_PATH}")
        add_library(xplm SHARED IMPORTED GLOBAL)
        add_library(xpwidgets SHARED IMPORTED GLOBAL)
        set_property(TARGET xplm PROPERTY IMPORTED_IMPLIB "${XPLM_LIBRARY}")
        set_property(TARGET xpwidgets PROPERTY IMPORTED_IMPLIB "${XPWIDGETS_LIBRARY}")
    endif()

    set(XPLM_LIBRARY "${XPLM_LIBRARY}" PARENT_SCOPE)
    set(XPWIDGETS_LIBRARY "${XPWIDGETS_LIBRARY}" PARENT_SCOPE)

    # For some reason this doesn't work. It's not worth wasting more time on fixing it, so
    # I'll manually include the directories below
    set_property(TARGET xplm APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/sdk/CHeaders/XPLM>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/sdk/CHeaders/Wrappers>
        $<INSTALL_INTERFACE:sdk/CHeaders/XPLM>
        $<INSTALL_INTERFACE:sdk/CHeaders/Wrappers>
    )
    set_property(TARGET xpwidgets APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/sdk/CHeaders/XPLM>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/sdk/CHeaders/Widgets>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/sdk/CHeaders/Wrappers>
        $<INSTALL_INTERFACE:sdk/CHeaders/XPLM>
        $<INSTALL_INTERFACE:sdk/CHeaders/Widgets>
        $<INSTALL_INTERFACE:sdk/CHeaders/Wrappers>
    )

    # for debug purposes:
    file(GENERATE OUTPUT "includes-xplm.txt" CONTENT "$<TARGET_PROPERTY:xplm,INTERFACE_INCLUDE_DIRECTORIES>\n")
    file(GENERATE OUTPUT "includes-xpwi.txt" CONTENT "$<TARGET_PROPERTY:xpwidgets,INTERFACE_INCLUDE_DIRECTORIES>\n")

elseif(UNIX)

    ## XPLANE SDK doesn't include any Linux static or shared library to be imported here
    ## -- documentation in [ www.xsquawkbox.net/xpsdk/mediawiki/BuildInstall ] specifies
    ## -- that in Linux, those symbols are left undefined and X-Plane itself defines them

endif()



# Set up an X-Plane plugin
function(add_xplane_plugin library_name ...)
    set(FILES ${ARGV})
    list(REMOVE_AT FILES 0)
    message(STATUS "Making X-Plane plugin called ${library_name} with (${FILES}) files")

    if(APPLE)
        add_executable(${library_name} ${FILES})
        target_compile_definitions(${library_name} PUBLIC -DAPL=1 -DIBM=0 -DLIN=0)
        set_target_properties(${library_name} PROPERTIES OUTPUT_NAME "mac.xpl")
    elseif(UNIX)
        add_library(${library_name} SHARED ${FILES})
        target_compile_definitions(${library_name} PUBLIC -DAPL=0 -DIBM=0 -DLIN=1)
        set_target_properties(${library_name} PROPERTIES OUTPUT_NAME "lin.xpl")
    elseif(WIN32)
        add_library(${library_name} MODULE ${FILES})
        target_compile_options(${library_name} PRIVATE "/MD$<$<CONFIG:Debug>:d>")
        target_compile_definitions(${library_name} PUBLIC -DAPL=0 -DIBM=1 -DLIN=0)
        target_compile_definitions(${library_name} PUBLIC -D_CRT_SECURE_NO_WARNINGS /bigobj)
        set_target_properties(${library_name} PROPERTIES OUTPUT_NAME "win.xpl")
    endif()

    target_compile_definitions(${library_name} PUBLIC -DXPLM200=1 -DXPLM210=1)

    #link flags
    if(APPLE)
        set_property(TARGET ${library_name} APPEND_STRING PROPERTY LINK_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fPIC -fvisibility=hidden -bundle")
    elseif(UNIX)
        set_property(TARGET ${library_name} APPEND_STRING PROPERTY LINK_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -rdynamic -nodefaultlibs -undefined_warning -fPIC -fvisibility=hidden")
    endif()

    if (APPLE OR WIN32)
        # Since XPLANE SDK only provides libraries for MAC & Windows, link againts them here.
        target_link_libraries(${library_name} PUBLIC ${XPLM_LIBRARY} ${XPWIDGETS_LIBRARY})
    endif(APPLE OR WIN32)

    target_include_directories(${library_name} PUBLIC "${XPLANE_INCLUDES_PATH}/XPLM" "${XPLANE_INCLUDES_PATH}/Widgets" "${XPLANE_INCLUDES_PATH}/Wrappers")
endfunction(add_xplane_plugin)
