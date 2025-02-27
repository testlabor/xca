
set(S "${PROJECT_SOURCE_DIR}/doc")
set(D "${PROJECT_BINARY_DIR}/doc")
file(MAKE_DIRECTORY "${D}")

find_program(SPHINX sphinx-build)
find_program(QTCOLLGEN NAMES qhelpgenerator qcollectiongenerator
                       HINTS ${_qt_bin_dir} ${_qt_bin_dir}/../libexec)

set(FILENAMES
	index commandline requests object-ids common-actions database options
	certificates revocationlist changelog introduction privatekey
	step-by-step certificate-input template smartcard miscellaneous
)

foreach(file ${FILENAMES})
  list(APPEND RST_SOURCE "${S}/rst/${file}.rst")
  list(APPEND RST_DEST "${D}/rst/${file}.rst")
  list(APPEND HTML_DEST "${D}/html/${file}.html")
  list(APPEND HTML_FILE "${file}.html")
  list(APPEND QTHELP_DEST "${D}/qthelp/${file}.html")
endforeach()

foreach(doc ${HTML_FILE} xca.qhc xca.qch)
  string(REPLACE "-" "" id ${doc})
  guid(${doc})
  list(APPEND DOC_COMPREFS "  <ComponentRef Id=\"Csphinx.${id}\"/>\n")
  list(APPEND DOC_COMPS "  <Component Id=\"Csphinx.${id}\" Guid=\"${GUID}\">
    <File Id=\"sphinx.${id}\" Source=\"${D}/qthelp/${doc}\" KeyPath=\"yes\"/>
  </Component>\n")
endforeach()

string(APPEND WIX_DOC_CONTENT
"<DirectoryRef Id=\"html\">\n" ${DOC_COMPS} "</DirectoryRef>\n"
"<ComponentGroup Id=\"GroupDocumentation\">\n" ${DOC_COMPREFS} "</ComponentGroup>"
)

WixFile(${D}/documentation.wxs ${WIX_DOC_CONTENT})

if(SPHINX)
  configure_file(${S}/conf.py.in ${D}/rst/conf.py)
  add_custom_command(
	OUTPUT ${HTML_DEST}
	COMMAND ${SPHINX} -q -b html ${D}/rst ${D}/html
	DEPENDS ${D}/rst/conf.py sphinx-src
	COMMENT "Create HTML documentation"
  )
  add_custom_target(sphinx-html DEPENDS ${HTML_DEST})
  add_custom_command(
	OUTPUT ${D}/qthelp/xca.qhcp ${D}/qthelp/xca.qhp ${QTHELP_DEST}
	COMMAND ${SPHINX} -q -b qthelp ${D}/rst ${D}/qthelp
	DEPENDS ${D}/rst/conf.py sphinx-src
	COMMENT "Create context sensitive help"
  )
  add_custom_target(sphinx-qthelp
	DEPENDS ${D}/qthelp/xca.qhcp ${D}/qthelp/xca.qhp ${QTHELP_DEST}
  )
  add_custom_command(
	OUTPUT ${D}/rst/database_schema.sql
	COMMAND ${CMAKE_COMMAND} -E make_directory ${D}/rst/_static
	COMMAND ${CMAKE_COMMAND}
		-D SRC=${PROJECT_SOURCE_DIR}/lib/database_schema.cpp
		-D DST=${D}/rst/database_schema.sql
		-P ${PROJECT_SOURCE_DIR}/cmake/database_schema.cmake
	DEPENDS ${PROJECT_SOURCE_DIR}/lib/database_schema.cpp
	COMMENT "Generating database schema SQL documentation"
  )
  add_custom_command(
	OUTPUT ${D}/rst.stamp
	COMMAND ${CMAKE_COMMAND} -E make_directory ${D}/rst/_static
	COMMAND ${CMAKE_COMMAND} -E copy_if_different
				${PROJECT_SOURCE_DIR}/img/bigcert.png
				${D}/rst/_static
	COMMAND ${CMAKE_COMMAND} -E copy_if_different
				${PROJECT_SOURCE_DIR}/COPYRIGHT
				${PROJECT_SOURCE_DIR}/changelog
				${RST_SOURCE} ${D}/rst
	DEPENDS ${PROJECT_SOURCE_DIR}/COPYRIGHT
		${PROJECT_SOURCE_DIR}/changelog
		${RST_SOURCE}
	COMMENT "Prepare Sphinx source directory"
  )
  add_custom_command(
	OUTPUT ${D}/rst/arguments.rst
	COMMAND xcadoc rst ${D}/rst/arguments.rst
	COMMENT "Generate 'rst' commandline documentation"
  )
  add_dependencies(${CMAKE_PROJECT_NAME} sphinx-html)
  target_sources(${CMAKE_PROJECT_NAME} PRIVATE ${HTML_DEST})
  add_custom_target(sphinx-src
	DEPENDS ${D}/rst.stamp
		${D}/rst/database_schema.sql
		${D}/rst/arguments.rst
  )

  if (UNIX)
    if (APPLE)
      execute_process(COMMAND ln -fs .doc/html/index.html ${D}/manual.html)
      install(FILES ${D}/manual.html DESTINATION .)
      set(HTML_INSTALL_DEST .doc/html)
      install(DIRECTORY ${D}/html DESTINATION .doc PATTERN ".*" EXCLUDE)
    else()
      install(DIRECTORY ${D}/html/ DESTINATION ${CMAKE_INSTALL_DOCDIR}
          PATTERN ".*" EXCLUDE
      )
    endif()
  endif()

  if(QTCOLLGEN)
    set(QTHELP_IDX ${D}/qthelp/xca.qhc ${D}/qthelp/xca.qch)
    add_custom_command(
	OUTPUT ${QTHELP_IDX}
	COMMAND ${QTCOLLGEN} ${D}/qthelp/xca.qhcp
	DEPENDS sphinx-qthelp
	COMMENT "Create context sensitive help index"
    )
    add_custom_target(sphinx-qtcollgen DEPENDS ${QTHELP_IDX})
    target_sources(${CMAKE_PROJECT_NAME} PRIVATE ${QTHELP_DEST} ${QTHELP_IDX})
    add_dependencies(${CMAKE_PROJECT_NAME} sphinx-qtcollgen)
    list(APPEND APP_RESOURCES ${QTHELP_DEST} ${QTHELP_IDX})

    if (UNIX AND NOT APPLE)
        install(FILES ${QTHELP_DEST} ${QTHELP_IDX}
                DESTINATION ${CMAKE_INSTALL_DATADIR}/${CMAKE_PROJECT_NAME}/html
        )
    elseif (WIN32)
        install(FILES ${QTHELP_DEST} ${QTHELP_IDX}
                DESTINATION ${RUNTIME_DEST}/html
        )
    endif()

  else(QTCOLLGEN)
    message(WARNING "Neither 'qhelpgenerator' nor 'qcollectiongenerator' found. In-app documentation disabed. ${_qt_bin_dir}")
  endif(QTCOLLGEN)
else(SPHINX)
  message(WARNING "sphinx-build binary not found. Documentation generation disabled")
endif(SPHINX)

if (NOT WIN32)
  add_custom_command(
	OUTPUT "${D}/xca.1.gz"
	COMMAND cat "${S}/xca.1.head" "${D}/xca.1.options" "${S}/xca.1.tail" | gzip -n9 > "${D}/xca.1.gz"
	DEPENDS "${S}/xca.1.head" "${S}/xca.1.tail"
		"${D}/xca.1.options"
	COMMENT "Compiling man page"
  )
  add_custom_command(
	OUTPUT "${D}/xca.1.options"
	COMMAND xcadoc man "${D}/xca.1.options"
	COMMENT "Generate 'man' commandline documentation"
  )
  add_custom_target(manpage ALL DEPENDS ${D}/xca.1.gz)
  if (NOT APPLE)
    install(FILES "${D}/xca.1.gz"
        DESTINATION ${CMAKE_INSTALL_MANDIR}/man1
    )
  endif()
  add_dependencies(${CMAKE_PROJECT_NAME} manpage)
endif()
