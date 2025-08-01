if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  message(FATAL_ERROR "cmake must be invoked with the top level directory")
endif()

add_executable(vncviewer
  fltk/Fl_Monitor_Arrangement.cxx
  fltk/Fl_Navigation.cxx
  fltk/theme.cxx
  BaseTouchHandler.cxx
  CConn.cxx
  DesktopWindow.cxx
  EmulateMB.cxx
  UserDialog.cxx
  ServerDialog.cxx
  ShortcutHandler.cxx
  Surface.cxx
  OptionsDialog.cxx
  PlatformPixelBuffer.cxx
  Viewport.cxx
  parameters.cxx
  touch.cxx
  MonitorIndicesParameter.cxx
  vncviewer.cxx)

if(WIN32 AND NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
  set_target_properties(vncviewer PROPERTIES WIN32_EXECUTABLE TRUE)
endif()

if(WIN32)
  # Since vncviewer.rc is generated, local includes will be looking
  # in the wrong directory. We need to help it out.
  target_include_directories(vncviewer PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
  configure_file(vncviewer.rc.in vncviewer.rc)
  target_sources(vncviewer PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/vncviewer.rc)
endif()

if(WIN32)
  target_sources(vncviewer PRIVATE Win32TouchHandler.cxx win32.c)
  target_sources(vncviewer PRIVATE KeyboardWin32.cxx keysym2ucs.c)
  target_sources(vncviewer PRIVATE Surface_Win32.cxx)
elseif(APPLE)
  target_sources(vncviewer PRIVATE cocoa.mm)
  target_sources(vncviewer PRIVATE KeyboardMacOS.mm osx_to_qnum.c keysym2ucs.c)
  target_sources(vncviewer PRIVATE Surface_OSX.cxx)
else()
  target_sources(vncviewer PRIVATE GestureHandler.cxx XInputTouchHandler.cxx)
  target_sources(vncviewer PRIVATE KeyboardX11.cxx xkb_to_qnum.c)
  target_sources(vncviewer PRIVATE Surface_X11.cxx)
endif()

target_include_directories(vncviewer SYSTEM PUBLIC ${FLTK_INCLUDE_DIR})
target_include_directories(vncviewer SYSTEM PUBLIC ${Intl_INCLUDE_DIR})
target_include_directories(vncviewer PUBLIC ${CMAKE_SOURCE_DIR}/common)
target_link_libraries(vncviewer core rfb network rdr)
target_link_libraries(vncviewer ${FLTK_LIBRARIES} ${Intl_LIBRARIES})

if(WIN32)
  target_link_libraries(vncviewer msimg32)
elseif(APPLE)
  target_link_libraries(vncviewer "-framework Cocoa")
  target_link_libraries(vncviewer "-framework Carbon")
  target_link_libraries(vncviewer "-framework IOKit")
else()
  target_link_libraries(vncviewer ${X11_Xi_LIB})

  if(X11_Xrandr_LIB)
    target_link_libraries(vncviewer ${X11_Xrandr_LIB})
  endif()
endif()

install(TARGETS vncviewer DESTINATION ${CMAKE_INSTALL_FULL_BINDIR})
if(UNIX)
  install(FILES vncviewer.man DESTINATION ${CMAKE_INSTALL_FULL_MANDIR}/man1 RENAME vncviewer.1)

  file(STRINGS ${CMAKE_SOURCE_DIR}/po/LINGUAS LINGUAS)
  set(po_FILES "")
  foreach(lang ${LINGUAS})
    list(APPEND po_FILES "${CMAKE_SOURCE_DIR}/po/${lang}.po")
  endforeach()

  configure_file(vncviewer.desktop.in.in vncviewer.desktop.in)
  find_program(INTLTOOL_MERGE_EXECUTABLE intltool-merge)
  if("${GETTEXT_VERSION_STRING}" VERSION_GREATER 0.18.99)
    add_custom_command(OUTPUT vncviewer.desktop
      COMMAND ${GETTEXT_MSGFMT_EXECUTABLE}
                --desktop --template vncviewer.desktop.in
                -d ${CMAKE_SOURCE_DIR}/po -o vncviewer.desktop
      DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/vncviewer.desktop.in
              ${po_FILES}
    )
  elseif(INTLTOOL_MERGE_EXECUTABLE)
    add_custom_command(OUTPUT vncviewer.desktop
      COMMAND sed -e 's/^Name/_Name/'
                -e 's/^GenericName/_GenericName/'
                -e 's/^Comment/_Comment/'
                vncviewer.desktop.in > vncviewer.desktop.intl
      COMMAND ${INTLTOOL_MERGE_EXECUTABLE}
                -d ${CMAKE_SOURCE_DIR}/po
                vncviewer.desktop.intl vncviewer.desktop
      DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/vncviewer.desktop.in
              ${po_FILES}
    )
  else()
    add_custom_command(OUTPUT vncviewer.desktop
      COMMAND cp vncviewer.desktop.in vncviewer.desktop
      DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/vncviewer.desktop.in
    )
  endif()
  add_custom_target(desktop ALL DEPENDS vncviewer.desktop)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/vncviewer.desktop DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/applications)

  if("${GETTEXT_VERSION_STRING}" VERSION_GREATER 0.19.6)
    add_custom_command(OUTPUT org.tigervnc.vncviewer.metainfo.xml
      COMMAND ${GETTEXT_MSGFMT_EXECUTABLE}
                --xml --template ${CMAKE_CURRENT_SOURCE_DIR}/org.tigervnc.vncviewer.metainfo.xml.in
                -d ${CMAKE_SOURCE_DIR}/po -o org.tigervnc.vncviewer.metainfo.xml
      DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/org.tigervnc.vncviewer.metainfo.xml.in
              ${po_FILES}
    )
  elseif(INTLTOOL_MERGE_EXECUTABLE)
    add_custom_command(OUTPUT org.tigervnc.vncviewer.metainfo.xml
      COMMAND sed -e 's@<name>@<_name>@\;s@</name>@</_name>@'
                  -e 's@<summary>@<_summary>@\;s@</summary>@</_summary>@'
                  -e 's@<caption>@<_caption>@\;s@</caption>@</_caption>@'
                  -e 's@<p>@<_p>@g\;s@</p>@</_p>@g'
                ${CMAKE_CURRENT_SOURCE_DIR}/org.tigervnc.vncviewer.metainfo.xml.in > org.tigervnc.vncviewer.metainfo.xml.intl
      COMMAND ${INTLTOOL_MERGE_EXECUTABLE}
                -x ${CMAKE_SOURCE_DIR}/po
                org.tigervnc.vncviewer.metainfo.xml.intl org.tigervnc.vncviewer.metainfo.xml
      DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/org.tigervnc.vncviewer.metainfo.xml.in
              ${po_FILES}
    )
  else()
    add_custom_command(OUTPUT org.tigervnc.vncviewer.metainfo.xml
      COMMAND cp ${CMAKE_CURRENT_SOURCE_DIR}/org.tigervnc.vncviewer.metainfo.xml.in org.tigervnc.vncviewer.metainfo.xml
      DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/org.tigervnc.vncviewer.metainfo.xml.in
    )
  endif()
  add_custom_target(appstream ALL DEPENDS org.tigervnc.vncviewer.metainfo.xml)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/org.tigervnc.vncviewer.metainfo.xml DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/metainfo)

  foreach(res 16 22 24 32 48 64 128)
    install(FILES ../media/icons/tigervnc_${res}.png DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/icons/hicolor/${res}x${res}/apps RENAME tigervnc.png)
  endforeach()
  install(FILES ../media/icons/tigervnc.svg DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/icons/hicolor/scalable/apps)
endif()
