#
# CMakeLists.txt
#
# Copyright (C) 2009-11 by RStudio, Inc.
#
# This program is licensed to you under the terms of version 3 of the
# GNU Affero General Public License. This program is distributed WITHOUT
# ANY EXPRESS OR IMPLIED WARRANTY, INCLUDING THOSE OF NON-INFRINGEMENT,
# MERCHANTABILITY OR FITNESS FOR A PARTICULAR PURPOSE. Please refer to the
# AGPL (http://www.gnu.org/licenses/agpl-3.0.txt) for more details.
#
#

project (SERVER)

add_subdirectory(pam)

# include files
file(GLOB_RECURSE SERVER_HEADER_FILES "*.h*")

# source files
set(SERVER_SOURCE_FILES
   ServerAppArmor.cpp
   ServerBrowser.cpp
   ServerMain.cpp
   ServerOffline.cpp
   ServerOptions.cpp
   ServerPAMAuth.cpp
   ServerREnvironment.cpp
   ServerSessionProxy.cpp
   auth/ServerAuthHandler.cpp
   auth/ServerSecureCookie.cpp
   auth/ServerSecureUriHandler.cpp
   auth/ServerValidateUser.cpp
   ${CMAKE_CURRENT_BINARY_DIR}/ServerAddins.cpp
)

# define core include dirs
set(CORE_INCLUDE_DIRS ${CORE_SOURCE_DIR}/include)

# include addins
if(RSTUDIO_ADDINS_PATH)
   # search for addins (then remove special core directory)
   file(GLOB RSTUDIO_ADDINS ${RSTUDIO_ADDINS_PATH}/*)
   list(REMOVE_ITEM RSTUDIO_ADDINS "core")

   # incorporate all addins found
   foreach(RSTUDIO_ADDIN ${RSTUDIO_ADDINS})
      set(SERVER_ADDIN_PATH  ${RSTUDIO_ADDIN}/server)
      if(EXISTS ${SERVER_ADDIN_PATH})
          # glob the addin header, source, and template files
         file(GLOB_RECURSE ADDIN_HEADER_FILES "${SERVER_ADDIN_PATH}/*.h*")
         list(APPEND SERVER_HEADER_FILES ${ADDIN_HEADER_FILES})
         file(GLOB_RECURSE ADDIN_SOURCE_FILES "${SERVER_ADDIN_PATH}/*.c*")
         list(APPEND SERVER_SOURCE_FILES ${ADDIN_SOURCE_FILES})
         file(GLOB_RECURSE ADDIN_TEMPLATE_FILES "${SERVER_ADDIN_PATH}/templates/*")
         list(APPEND SERVER_ADDIN_TEMPLATE_FILES ${ADDIN_TEMPLATE_FILES})
         # generate an initialize call for the addin
         get_filename_component(ADDIN_NAME ${RSTUDIO_ADDIN} NAME)
         set(SERVER_ADDIN_DECLARATIONS
            "${SERVER_ADDIN_DECLARATIONS}namespace ${ADDIN_NAME} { Error initialize(); }\n" )
         set(SERVER_ADDIN_INITIALIZATIONS
            "${SERVER_ADDIN_INITIALIZATIONS}(${ADDIN_NAME}::initialize) ")
      endif()
   endforeach()

   # add to core include dirs if appropriate
   set(CORE_ADDINS_INCLUDE_DIR ${RSTUDIO_ADDINS_PATH}/core/include)
   if(EXISTS ${CORE_ADDINS_INCLUDE_DIR})
      list(APPEND CORE_INCLUDE_DIRS ${CORE_ADDINS_INCLUDE_DIR})
   endif()

endif()

# always configure the addins bootstrap file
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/ServerAddins.cpp.in
               ${CMAKE_CURRENT_BINARY_DIR}/ServerAddins.cpp)

# configure template files into the www directory
foreach(SERVER_ADDIN_TEMPLATE_FILE ${SERVER_ADDIN_TEMPLATE_FILES})
   get_filename_component(TEMPLATE_FILE_NAME ${SERVER_ADDIN_TEMPLATE_FILE} NAME)
   configure_file(${SERVER_ADDIN_TEMPLATE_FILE}
                  "${CMAKE_CURRENT_SOURCE_DIR}/../../gwt/www/templates/addins/${TEMPLATE_FILE_NAME}"
                  COPYONLY)
endforeach()

# set flag for platforms that are known to be able to call PAM routines
# without requring restorePriv (to avoid geteuid != getuid)
if("${RSTUDIO_LSB_RELEASE}" STREQUAL "ubuntu" OR
   "${RSTUDIO_LSB_RELEASE}" STREQUAL "debian")
   set(HAVE_PAM_REQUIRES_RESTORE_PRIV FALSE)
else()
   set(HAVE_PAM_REQUIRES_RESTORE_PRIV TRUE)
endif()

# look for setresuid
check_function_exists(setresuid HAVE_SETRESUID)

# generate config file
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.in
               ${CMAKE_CURRENT_BINARY_DIR}/config.h)

# require openssl (for crypto)
find_package(OpenSSL REQUIRED QUIET)

# include directories and libraries
set(SERVER_SYSTEM_INCLUDE_DIRS ${OPENSSL_INCLUDE_DIRS})
set(SERVER_SYSTEM_LIBRARIES ${OPENSSL_LIBRARIES})

set(SERVER_SOURCE_FILES ${SERVER_SOURCE_FILES}
   util/system/PosixCrypto.cpp
   util/system/PosixSystem.cpp
   util/system/PosixUser.cpp
)

# set include directories
include_directories(
   include
   ${CMAKE_CURRENT_BINARY_DIR}
   ${Boost_INCLUDE_DIRS}
   ${SERVER_SYSTEM_INCLUDE_DIRS}
   ${CORE_INCLUDE_DIRS}
   ${SESSION_SOURCE_DIR}/include
)

# define executable
add_executable(rserver ${SERVER_SOURCE_FILES} ${SERVER_HEADER_FILES})

# set link dependencies
target_link_libraries(rserver 
   rstudio-core
   ${SERVER_SYSTEM_LIBRARIES}
)

# install binary
install(TARGETS rserver DESTINATION ${RSTUDIO_INSTALL_BIN})

if (UNIX AND NOT APPLE)

   # install configured admin script
   set(RSERVER_ADMIN_SCRIPT "extras/admin/rstudio-server")
   configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${RSERVER_ADMIN_SCRIPT}.in
                  ${CMAKE_CURRENT_BINARY_DIR}/${RSERVER_ADMIN_SCRIPT})
   install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/${RSERVER_ADMIN_SCRIPT}
                    DESTINATION ${RSTUDIO_INSTALL_BIN})

   # install configured debian init.d script
   set(RSERVER_INITD_DEBIAN_DIR "extras/init.d/debian")
   set(RSERVER_INITD_DEBIAN_SCRIPT "${RSERVER_INITD_DEBIAN_DIR}/rstudio-server")
   configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${RSERVER_INITD_DEBIAN_SCRIPT}.in
                  ${CMAKE_CURRENT_BINARY_DIR}/${RSERVER_INITD_DEBIAN_SCRIPT})
   install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/${RSERVER_INITD_DEBIAN_SCRIPT}
           DESTINATION ${RSERVER_INITD_DEBIAN_DIR})

   # install configured redhat init.d script
   set(RSERVER_INITD_REDHAT_DIR "extras/init.d/redhat")
   set(RSERVER_INITD_REDHAT_SCRIPT "${RSERVER_INITD_REDHAT_DIR}/rstudio-server")
   configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${RSERVER_INITD_REDHAT_SCRIPT}.in
                  ${CMAKE_CURRENT_BINARY_DIR}/${RSERVER_INITD_REDHAT_SCRIPT})
   install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/${RSERVER_INITD_REDHAT_SCRIPT}
           DESTINATION ${RSERVER_INITD_REDHAT_DIR})

   # install configured suse init.d script
   set(RSERVER_INITD_SUSE_DIR "extras/init.d/suse")
   set(RSERVER_INITD_SUSE_SCRIPT "${RSERVER_INITD_SUSE_DIR}/rstudio-server")
   configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${RSERVER_INITD_SUSE_SCRIPT}.in
                  ${CMAKE_CURRENT_BINARY_DIR}/${RSERVER_INITD_SUSE_SCRIPT})
   install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/${RSERVER_INITD_SUSE_SCRIPT}
           DESTINATION ${RSERVER_INITD_SUSE_DIR})

   # install pam profile
   set(RSERVER_PAM_DIR "extras/pam")
   set(RSERVER_PAM_PROFILE "${RSERVER_PAM_DIR}/rstudio")
   install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/${RSERVER_PAM_PROFILE}
           DESTINATION ${RSERVER_PAM_DIR})

   # install configured apparmor profile
   set(RSERVER_APPARMOR_DIR "extras/apparmor")
   set(RSERVER_APPARMOR_PROFILE "${RSERVER_APPARMOR_DIR}/rstudio-server")
   configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${RSERVER_APPARMOR_PROFILE}.in
                  ${CMAKE_CURRENT_BINARY_DIR}/${RSERVER_APPARMOR_PROFILE})
   install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${RSERVER_APPARMOR_PROFILE}
           DESTINATION ${RSERVER_APPARMOR_DIR})
   install(PROGRAMS ${CMAKE_CURRENT_SOURCE_DIR}/${RSERVER_APPARMOR_DIR}/apparmor-profile-load
           DESTINATION ${RSERVER_APPARMOR_DIR})

   # install configured upstart profile
   set(RSERVER_UPSTART_DIR "extras/upstart")
   set(RSERVER_UPSTART_PROFILE "${RSERVER_UPSTART_DIR}/rstudio-server.conf")
   configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${RSERVER_UPSTART_PROFILE}.in
                  ${CMAKE_CURRENT_BINARY_DIR}/${RSERVER_UPSTART_PROFILE})
   install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${RSERVER_UPSTART_PROFILE}
           DESTINATION ${RSERVER_UPSTART_DIR})

endif()
