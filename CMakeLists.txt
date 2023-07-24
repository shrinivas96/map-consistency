CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
PROJECT(consist)

# SET(CMAKE_BUILD_TYPE Release)
SET(CMAKE_BUILD_TYPE Debug)

# not including this line gave the error: `undefined reference to symbol 'pthread_join@@GLIBC_2.2.5'` AND `DSO missing from command line`
# source to fix this line: https://stackoverflow.com/a/61439452/6609148
# the better way to do it: https://stackoverflow.com/a/29871891/6609148
# SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread") 
# set(THREADS_PREFER_PTHREAD_FLAG ON)

SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

FIND_PACKAGE(Threads REQUIRED)
FIND_PACKAGE(BLAS REQUIRED)
FIND_PACKAGE(Eigen3 REQUIRED)
FIND_PACKAGE(Cholmod REQUIRED)
FIND_PACKAGE(G2O REQUIRED)          # returns system g2o somehow
# add_subdirectory(libs/g2o)        # to allow it to find g2o in the same project; does NOT work completely for now
# Called by FindQGLViewer
# FIND_PACKAGE(Qt4)
FIND_PACKAGE(QGLViewer)
FIND_PACKAGE(OpenGL)
FIND_PACKAGE(Doxygen)

# set(DEBUG_MSGS TRUE)
if (DEBUG_MSGS)
    message(STATUS "!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!")

    # if using add_subdirectory, g2o/CMakeLists.txt should set these varaibles.
    # i think these should be linked to the targets to make everything work
    # and the binary dir should be added to the include_directories()
    message(STATUS ${g2o_LIBRARY_OUTPUT_DIRECTORY})
    message(STATUS ${g2o_BINARY_DIR})

    message(STATUS "TARGETS ${G2O_CORE_LIBRARY}, ${G2O_STUFF_LIBRARY}, ${G2O_TYPES_SLAM2D}, ${G2O_TYPES_DATA}, ${G2O_SOLVER_CHOLMOD},")

    # prints yes if found. if this is not found see FindG2O.cmake file once
    message(STATUS "G2O FOUND: ")
    message(STATUS ${G2O_FOUND})

    # location of where G2O was found
    message(STATUS "G2O INCLUDE DIRS: ")
    message(STATUS ${G2O_INCLUDE_DIRS}) 

    message(STATUS "G2O Link Libraries: ")
    message(STATUS ${G2O_LINK_LIBRARIES}) 

    # this is where g2o should have been found
    message(STATUS "location")
    message(STATUS ${PROJECT_SOURCE_DIR}/libs/g2o)

    message(STATUS "G2O Solvers: ")
    message(STATUS ${G2O_SOLVER_CHOLMOD})
    message(STATUS "!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!")
endif ()

IF(QGLVIEWER_LIBRARY AND QT5_FOUND AND OPENGL_FOUND)
	MESSAGE(STATUS "Enabling GUI support")
	SET(WITH_GUI TRUE)
ELSE()
	MESSAGE(STATUS "Disabling GUI support")
	SET(WITH_GUI FALSE)
ENDIF()

INCLUDE_DIRECTORIES(${EIGEN3_INCLUDE_DIR})
INCLUDE_DIRECTORIES(${CHOLMOD_INCLUDE_DIR})
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/src)
INCLUDE_DIRECTORIES(${G2O_INCLUDE_DIRS})                    # this variable is set by FindG2o.cmake; currently finds the ros installation
# INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/libs/g2o)       # when it was unable to find the headers.
# INCLUDE_DIRECTORIES(${PROJECT_BINARY_DIR})
# INCLUDE_DIRECTORIES(${g2o_BINARY_DIR})                    # should include when add_subdirectory for g2o is the case

IF(DOXYGEN_FOUND)
    MESSAGE(STATUS "Enabling documentation build")
    CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile @ONLY)
    ADD_CUSTOM_TARGET(doc
        ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        COMMENT "Generating API documentation with Doxygen" VERBATIM)
ELSE(DOXYGEN_FOUND)
    MESSAGE(STATUS "Disabling documentation build")
ENDIF(DOXYGEN_FOUND)

IF(WITH_GUI)
    SET(QT_USE_QTOPENGL TRUE)
    SET(QT_USE_QTXML TRUE)
	INCLUDE(${QT_USE_FILE})
    ADD_DEFINITIONS(${QT_DEFINITIONS})
    INCLUDE_DIRECTORIES(${OPENGL_INCLUDE_DIR})
	INCLUDE_DIRECTORIES(${QGLVIEWER_INCLUDE_DIR})
	INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})
    SET(GUI_LIBRARIES
        ${QT_LIBRARIES}
        ${QGLVIEWER_LIBRARY}
        ${OPENGL_LIBRARIES})
ENDIF(WITH_GUI)

ADD_LIBRARY(consist STATIC
    src/consist/gridbisectionoptimizer.cpp
    src/consist/hypothesistester.cpp
    src/consist/inconsistencyse2.cpp
    src/consist/mapquality.cpp
    src/consist/probability.cpp
    src/consist/quadrilateral.cpp
    src/consist/support.cpp
    src/consist/visibility.cpp)

SET(shared_SOURCES
	src/shared/argumentparser.cpp)
	
SET(optimizer_SOURCES
	${shared_SOURCES}
    src/optimizer/optimizeroptions.cpp
    src/optimizer/optimizer.cpp)
    
SET(tester_SOURCES
	${shared_SOURCES}
    src/tester/tester.cpp
    src/tester/testeroptions.cpp)
	
IF(WITH_GUI)
	ADD_DEFINITIONS(-DGUI_SUPPORT)
	SET(tester_SOURCES
		${tester_SOURCES}
		src/tester/mapviewer.cpp
		src/tester/testerwindow.cpp)
	QT4_WRAP_UI(tester_HEADERS_FORMS
		forms/testerwindow.ui)
	QT4_WRAP_CPP(tester_HEADERS_MOC
		src/tester/mapviewer.h
		src/tester/testerwindow.h)
ENDIF(WITH_GUI)

ADD_EXECUTABLE(optimizer
    ${optimizer_SOURCES})
    
ADD_EXECUTABLE(tester
    ${tester_SOURCES}
    ${tester_HEADERS_FORMS}
    ${tester_HEADERS_MOC})

TARGET_LINK_LIBRARIES(optimizer
	consist
    ${BLAS_LIBRARIES}
    ${CHOLMOD_LIBRARY}
    ${G2O_CORE_LIBRARY}
    ${G2O_STUFF_LIBRARY}
    ${G2O_TYPES_SLAM2D}
    ${G2O_TYPES_DATA}
    ${G2O_SOLVER_CHOLMOD}
    Threads::Threads)          # Threads::Threads or ${CMAKE_THREAD_LIBS_INIT}
    
TARGET_LINK_LIBRARIES(tester
	consist
    ${BLAS_LIBRARIES}
    ${GUI_LIBRARIES}
    ${CHOLMOD_LIBRARY}
    ${G2O_CORE_LIBRARY}
    ${G2O_STUFF_LIBRARY}
    ${G2O_TYPES_SLAM2D}
    ${G2O_TYPES_DATA}
    ${G2O_SOLVER_CHOLMOD}
    Threads::Threads)          # Threads::Threads or ${CMAKE_THREAD_LIBS_INIT}
