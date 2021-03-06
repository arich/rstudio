
# append our module directory to path
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules")

# cpack variables
set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}")
set(CPACK_NSIS_INSTALLED_ICON_NAME "bin\\rstudio.exe")
set(CPACK_PACKAGE_INSTALL_REGISTRY_KEY "RStudio")
set(CPACK_PACKAGE_EXECUTABLES "rstudio" "RStudio")
set(CPACK_INCLUDE_TOPLEVEL_DIRECTORY 0)

# registry keys
set(CPACK_NSIS_EXTRA_INSTALL_COMMANDS "
   WriteRegStr HKLM 'Software\\\\Microsoft\\\\Windows\\\\CurrentVersion\\\\App Paths\\\\rstudio.exe' '' '$INSTDIR\\\\bin\\\\rstudio.exe'

   WriteRegStr HKCR 'Applications\\\\rstudio.exe\\\\shell\\\\open\\\\command' '' '$INSTDIR\\\\bin\\\\rstudio.exe \\\"%1\\\"'

   WriteRegStr HKCR 'Applications\\\\rstudio.exe\\\\SupportedTypes' '.R' ''
   WriteRegStr HKCR 'Applications\\\\rstudio.exe\\\\SupportedTypes' '.RData' ''
   WriteRegStr HKCR 'Applications\\\\rstudio.exe\\\\SupportedTypes' '.rda' ''
   WriteRegStr HKCR 'Applications\\\\rstudio.exe\\\\SupportedTypes' '.Rd' ''
   WriteRegStr HKCR 'Applications\\\\rstudio.exe\\\\SupportedTypes' '.Rnw' ''

   WriteRegStr HKCR '.R\\\\OpenWithList\\\\rstudio.exe' '' ''
   WriteRegStr HKCR '.RData\\\\OpenWithList\\\\rstudio.exe' '' ''
   WriteRegStr HKCR '.rda\\\\OpenWithList\\\\rstudio.exe' '' ''
   WriteRegStr HKCR '.Rd\\\\OpenWithList\\\\rstudio.exe' '' ''
   WriteRegStr HKCR '.Rnw\\\\OpenWithList\\\\rstudio.exe' '' ''
   WriteRegStr HKCR '.tex\\\\OpenWithList\\\\rstudio.exe' '' ''

   System::Call 'Shell32::SHChangeNotify(i 0x8000000, i 0, i 0, i 0)'
")

set(CPACK_NSIS_EXTRA_UNINSTALL_COMMANDS "
   DeleteRegKey HKLM 'Software\\\\Microsoft\\\\Windows\\\\CurrentVersion\\\\App Paths\\\\rstudio.exe'
   DeleteRegKey HKCR 'Applications\\\\rstudio.exe'
   DeleteRegKey HKCR '.R\\\\OpenWithList\\\\rstudio.exe'
   DeleteRegKey HKCR '.RData\\\\OpenWithList\\\\rstudio.exe'
   DeleteRegKey HKCR '.rda\\\\OpenWithList\\\\rstudio.exe'
   DeleteRegKey HKCR '.Rd\\\\OpenWithList\\\\rstudio.exe'
   DeleteRegKey HKCR '.Rnw\\\\OpenWithList\\\\rstudio.exe'
   DeleteRegKey HKCR '.tex\\\\OpenWithList\\\\rstudio.exe'
#")

# build installer
include(CPack)

