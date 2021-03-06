#/*============================================================================
#
#  BARD: Basic Augmented Reality Demo.
#
#  Copyright (c) University College London (UCL). All rights reserved.
#
#  This software is distributed WITHOUT ANY WARRANTY; without even
#  the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR
#  PURPOSE.
#
#  See LICENSE.txt in the top level directory for details.
#
#============================================================================*/

set(VIDEO_SRCS
  bardFileIO.cxx
  bardMaths.cxx
  bardVideoSourceInterface.cxx
  bardOpenCVVideoSource.cxx
  bardCalibratedCamera.cxx
  bardTagData.cxx
  bardTagProcessingInterface.cxx
  bardArucoProcessor.cxx
  bardModelData.cxx
  bardRegistrationInterface.cxx
  bardPnPRegistration.cxx
  bardTrackingModelInterface.cxx
  bardTrackingModelData.cxx
  bardVTKModelInterface.cxx
  bardVTKModelPipeline.cxx
)

SET(QT_SOURCES_CPP
bardMainWindow.cxx
bardMainRenderingWidget.cxx
)

SET(QT_SOURCES_H
bardMainWindow.h
bardMainRenderingWidget.h
)

SET(QT_SOURCES_UI
bardMainWindow.ui
)

# Generate QT stuff.
if(DESIRED_QT_VERSION MATCHES 5)
  QT5_WRAP_UI(QT_SOURCES_UI_H ${QT_SOURCES_UI} )
  QT5_WRAP_CPP(QT_MOC_FILES ${QT_SOURCES_H})
else()
  QT4_WRAP_UI(QT_SOURCES_UI_H ${QT_SOURCES_UI} )
  QT4_WRAP_CPP(QT_MOC_FILES ${QT_SOURCES_H})
endif()

SOURCE_GROUP("Generated Files" FILES ${QT_MOC_FILES})
include_directories(${CMAKE_CURRENT_BINARY_DIR})

foreach(BARD_APP bard bardGrabber bardCameraCalibration bardPivotCalibration bardProcrustes  )
  add_executable(${BARD_APP} ${BARD_APP}.cxx ${VIDEO_SRCS} ${QT_SOURCES_CPP} ${QT_SOURCES_H} ${QT_SOURCES_UI_H} ${QT_MOC_FILES})
  target_link_libraries(${BARD_APP} ${ALL_LIBS})
endforeach()
