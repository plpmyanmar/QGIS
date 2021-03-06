########################################################
# Files

SET(PG_SRCS
  qgspostgresprovider.cpp
  qgspostgresconn.cpp
  qgspostgresconnpool.cpp
  qgspostgresdataitems.cpp
  qgspostgresfeatureiterator.cpp
  qgspostgresprojectstorage.cpp
  qgspostgrestransaction.cpp
  qgspgtablemodel.cpp
  qgscolumntypethread.cpp
  qgspostgresexpressioncompiler.cpp
  qgspostgreslistener.cpp
  qgspostgresproviderconnection.cpp
)

IF (WITH_GUI)
  SET(PG_GUI_SRCS
    qgspostgresprovidergui.cpp
    qgspostgresdataitemguiprovider.cpp
    qgspgsourceselect.cpp
    qgspgnewconnection.cpp
    qgspostgresprojectstoragedialog.cpp
  )
ENDIF ()

SET(PG_HDRS
  qgspostgresexpressioncompiler.h
  qgspostgresproviderconnection.h
)

########################################################
# Build

INCLUDE_DIRECTORIES(
  ${CMAKE_SOURCE_DIR}/src/core
  ${CMAKE_SOURCE_DIR}/src/core/auth
  ${CMAKE_SOURCE_DIR}/src/core/expression
  ${CMAKE_SOURCE_DIR}/src/core/geometry
  ${CMAKE_SOURCE_DIR}/src/core/metadata
  ${CMAKE_SOURCE_DIR}/src/core/symbology
  ${CMAKE_SOURCE_DIR}/src/core/raster
  ${CMAKE_SOURCE_DIR}/src/gui
  ${CMAKE_SOURCE_DIR}/src/gui/auth
  ${CMAKE_SOURCE_DIR}/external
  ${CMAKE_SOURCE_DIR}/external/nlohmann

  ${CMAKE_BINARY_DIR}/src/core
  ${CMAKE_BINARY_DIR}/src/gui
  ${CMAKE_BINARY_DIR}/src/ui
)

INCLUDE_DIRECTORIES(SYSTEM
  ${POSTGRES_INCLUDE_DIR}
  ${QCA_INCLUDE_DIR}
  ${QTKEYCHAIN_INCLUDE_DIR}
  ${GDAL_INCLUDE_DIR}
  ${SPATIALINDEX_INCLUDE_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}
)

IF (WITH_GUI)
  INCLUDE_DIRECTORIES(SYSTEM
    ${QSCINTILLA_INCLUDE_DIR}
  )
ENDIF ()

# static library
ADD_LIBRARY (postgresprovider_a STATIC ${PG_SRCS} ${PG_HDRS})
TARGET_LINK_LIBRARIES (postgresprovider_a
  ${POSTGRES_LIBRARY}
  ${Qt5Xml_LIBRARIES}
  ${Qt5Core_LIBRARIES}
  ${Qt5Svg_LIBRARIES}
  ${Qt5Network_LIBRARIES}
  ${Qt5Sql_LIBRARIES}
  ${Qt5Concurrent_LIBRARIES}
)

IF (WITH_GUI)
  ADD_LIBRARY(postgresprovider_gui_a STATIC ${PG_GUI_SRCS})
  TARGET_LINK_LIBRARIES(postgresprovider_gui_a
    ${POSTGRES_LIBRARY}
    ${Qt5Xml_LIBRARIES}
    ${Qt5Core_LIBRARIES}
    ${Qt5Svg_LIBRARIES}
    ${Qt5Network_LIBRARIES}
    ${Qt5Sql_LIBRARIES}
    ${Qt5Concurrent_LIBRARIES}
    ${Qt5Gui_LIBRARIES}
    ${Qt5Widgets_LIBRARIES}
  )
  ADD_DEPENDENCIES(postgresprovider_gui_a ui)
ENDIF (WITH_GUI)

#################################################################
# Postgres Raster

SET(PGRASTER_SRCS
  raster/qgspostgresrasterprovider.cpp
  raster/qgspostgresrastershareddata.cpp
  raster/qgspostgresrasterutils.cpp
  qgspostgresconn.cpp
  qgspostgresconnpool.cpp
)

# static library
ADD_LIBRARY (postgresrasterprovider_a STATIC ${PGRASTER_SRCS} ${PG_HDRS})
TARGET_LINK_LIBRARIES (postgresrasterprovider_a
    ${POSTGRES_LIBRARY}
    ${Qt5Xml_LIBRARIES}
    ${Qt5Core_LIBRARIES}
    ${Qt5Svg_LIBRARIES}
    ${Qt5Network_LIBRARIES}
    ${Qt5Sql_LIBRARIES}
    ${Qt5Concurrent_LIBRARIES}
    ${Qt5Gui_LIBRARIES}
    ${Qt5Widgets_LIBRARIES}
)

#################################################################

IF (FORCE_STATIC_PROVIDERS)
  # for (external) mobile apps to be able to pick up provider for linking
  INSTALL (TARGETS postgresprovider_a ARCHIVE DESTINATION ${QGIS_PLUGIN_DIR})
  INSTALL (TARGETS postgresrasterprovider_a ARCHIVE DESTINATION ${QGIS_PLUGIN_DIR})
  IF (WITH_GUI)
    INSTALL (TARGETS postgresprovider_gui_a ARCHIVE DESTINATION ${QGIS_PLUGIN_DIR})
  ENDIF (WITH_GUI)
ELSE (FORCE_STATIC_PROVIDERS)
  # dynamically loaded module
  ADD_LIBRARY(postgresprovider MODULE ${PG_SRCS} ${PG_GUI_SRCS} ${PG_HDRS})
  ADD_LIBRARY(postgresrasterprovider MODULE ${PGRASTER_SRCS} ${PG_HDRS})

  TARGET_LINK_LIBRARIES(postgresprovider
    ${POSTGRES_LIBRARY}
    qgis_core
  )

  TARGET_LINK_LIBRARIES(postgresrasterprovider
    ${POSTGRES_LIBRARY}
    qgis_core
  )

  IF (WITH_GUI)
    TARGET_LINK_LIBRARIES (postgresprovider
      qgis_gui
    )
    ADD_DEPENDENCIES(postgresprovider ui)    
  ENDIF (WITH_GUI)


  # clang-tidy
  IF(CLANG_TIDY_EXE)
    SET_TARGET_PROPERTIES(
      postgresprovider PROPERTIES
      CXX_CLANG_TIDY "${DO_CLANG_TIDY}"
    )
    SET_TARGET_PROPERTIES(
      postgresrasterprovider PROPERTIES
      CXX_CLANG_TIDY "${DO_CLANG_TIDY}"
    )
  ENDIF(CLANG_TIDY_EXE)

  INSTALL(TARGETS postgresprovider
    RUNTIME DESTINATION ${QGIS_PLUGIN_DIR}
    LIBRARY DESTINATION ${QGIS_PLUGIN_DIR})

  INSTALL(TARGETS postgresrasterprovider
    RUNTIME DESTINATION ${QGIS_PLUGIN_DIR}
    LIBRARY DESTINATION ${QGIS_PLUGIN_DIR})


ENDIF (FORCE_STATIC_PROVIDERS)
