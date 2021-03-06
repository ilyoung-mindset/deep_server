cmake_minimum_required(VERSION 2.8.7)
set( CMAKE_VERBOSE_MAKEFILE off )

if(MSVC)
    # CMake 3.4 introduced a WINDOWS_EXPORT_ALL_SYMBOLS target property that makes it possible to
    # build shared libraries without using the usual declspec() decoration.
    # See: https://blog.kitware.com/create-dlls-on-windows-without-declspec-using-new-cmake-export-all-feature/
    # and https://cmake.org/cmake/help/v3.5/prop_tgt/WINDOWS_EXPORT_ALL_SYMBOLS.html
    # for details.
    cmake_minimum_required(VERSION 3.4)
    set( CMAKE_BUILD_TYPE Debug )
else()
    set( CMAKE_BUILD_TYPE Release )
endif()

if(POLICY CMP0046)
    cmake_policy(SET CMP0046 NEW)
endif()
if(POLICY CMP0054)
    cmake_policy(SET CMP0054 NEW)
endif()

# ---[ deep_server project
project(deep_server C CXX)

set(DEEP_SRCS
    Depends/caffe/frcnn_wrapper.cpp
    Depends/yolo/yolo_wrapper.cpp
    Depends/tensorflow/tf_wrapper.cpp
    src/utils.cpp
    src/actorlogger.cpp
    src/base64.cpp
    src/caffe_process.cpp
    src/cv_process.cpp
    src/deep_server.cpp
    src/http_broker.cpp
    src/http_parser.cpp
    src/parserhelper.cpp
    src/tcp_broker.cpp
    src/yolo_process.cpp
    src/tf_process.cpp
    )

add_executable(deep_server ${DEEP_SRCS}) 

# ---[ deep_server version
set(DEEP_TARGET_VERSION "1.0.0-rc5" CACHE STRING "Deep logical version")
set(DEEP_TARGET_SOVERSION "1.0.0-rc5" CACHE STRING "Deep soname version")
add_definitions(-DDEEP_VERSION=${DEEP_TARGET_VERSION} -DGLOG_NO_ABBREVIATED_SEVERITIES)

# ---[ Using cmake scripts and modules
list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/Modules)

include(cmake/CompilerCheck.cmake)
include(cmake/Utils.cmake)
include(cmake/Summary.cmake)
include(cmake/ConfigGen.cmake)

if(MSVC)
    deep_option(CPU_ONLY  "Build deep with CUDA support" ON) 
    deep_option(USE_CUDA  "Build deep without CUDA support" OFF) 
    #add_definitions(-DCPU_ONLY -DBOOST_ALL_NO_LIB)
    add_definitions(-DUSE_CUDA -DBOOST_ALL_NO_LIB)
    add_definitions(-DYOLO_GPU)
else()
    deep_option(CPU_ONLY  "Build deep without CUDA support" OFF)
    deep_option(USE_CUDA  "Build deep with CUDA support" ON) 
    add_definitions(-DUSE_CUDA)
    add_definitions(-DYOLO_GPU)
endif()

deep_option(USE_CUDNN "Build deep with cuDNN library support" ON IF NOT CPU_ONLY)
deep_option(USE_OPENCV "Build with OpenCV support" ON)
deep_option(USE_OPENMP "Link with OpenMP (when your BLAS wants OpenMP and you get linker errors)" OFF)
deep_option(USE_NCCL "Build deep with NCCL library support" OFF)

include(cmake/Cuda.cmake)

if(MSVC)
    #set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "c:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/")
    include("c:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/caffe-builder-config.cmake")
endif()

include("cmake/External/glog.cmake")
include_directories(${GLOG_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} ${GLOG_LIBRARIES} )

# ---[ Google-gflags
include("cmake/External/gflags.cmake")
include_directories(${GFLAGS_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} ${GFLAGS_LIBRARIES} )

# ---[ Flags
if(UNIX OR APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -Wall")
endif()

if(USE_libstdcpp)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libstdc++")
    message("-- Warning: forcing libstdc++ (controlled by USE_libstdcpp option in cmake)")
endif()

# ---[ Warnings
deep_warnings_disable(CMAKE_CXX_FLAGS -Wno-sign-compare -Wno-uninitialized)

# ---[ Includes
#set(Deep_INCLUDE_DIR ${PROJECT_SOURCE_DIR}/include)
set(deep_server_SRC_DIR ${PROJECT_SOURCE_DIR}/src)
include_directories(${PROJECT_BINARY_DIR})
include_directories(${PROJECT_SOURCE_DIR}/Depends/caffe)
include_directories(${PROJECT_SOURCE_DIR}/Depends/yolo)
include_directories(${PROJECT_SOURCE_DIR}/Depends/tensorflow)

if(MSVC)
    set(TENSORFLOW_FOLDER c:/Users/Administrator/Desktop/tensorflow)
else()
    set(TENSORFLOW_FOLDER /data/pzl/tensorflow)
endif() 
include_directories(${TENSORFLOW_FOLDER}) 

set(CMAKE_POSITION_INDEPENDENT_CODE ON)
add_definitions(-DEIGEN_AVOID_STL_ARRAY)
if(WIN32)
  add_definitions(-DNOMINMAX -D_WIN32_WINNT=0x0A00 -DLANG_CXX11 -DCOMPILER_MSVC)
  add_definitions(-DWIN32 -DOS_WIN -D_MBCS -DWIN64 -DWIN32_LEAN_AND_MEAN -DNOGDI -DPLATFORM_WINDOWS)
  add_definitions(-DTENSORFLOW_USE_EIGEN_THREADPOOL -DEIGEN_HAS_C99_MATH)
  add_definitions(-DTF_COMPILE_LIBRARY)
  add_definitions(/bigobj /nologo /EHsc /GF /FC /MP /Gm-)
  # Suppress warnings to reduce build log size.
  add_definitions(/wd4267 /wd4244 /wd4800 /wd4503 /wd4554 /wd4996 /wd4348 /wd4018)
  add_definitions(/wd4099 /wd4146 /wd4267 /wd4305 /wd4307)
  add_definitions(/wd4715 /wd4722 /wd4723 /wd4838 /wd4309 /wd4334)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")
  set(CMAKE_CXX_FLAGS_DEBUG "/D_DEBUG /MDd /Ob0")
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /D_ITERATOR_DEBUG_LEVEL=0")
  set(CMAKE_CXX_FLAGS_MINSIZEREL "${CMAKE_CXX_FLAGS_MINSIZEREL} /D_ITERATOR_DEBUG_LEVEL=0")
  set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} /D_ITERATOR_DEBUG_LEVEL=0")
endif() 

if(MSVC)
    include_directories("d:/work/caffe-windows/caffe/include/")
    include_directories("d:/work/caffe-windows/caffe/build/include/")
    include_directories("d:/work/deep_server/3rd/actor-framework/libcaf_io/")
    include_directories("d:/work/deep_server/3rd/actor-framework/libcaf_core/")
    include_directories("d:/work/deep_server/3rd/jsoncpp-master/include/")
    include_directories("d:/work/yolo_centos/src/")
    include_directories("d:/work/yolo_centos/3rdparty/include/")
    include_directories(${PROJECT_SOURCE_DIR}/Depends/tensorflow)
    link_directories("d:/work/deep_server/3rd/jsoncpp-master/build/src/lib_json/${CMAKE_BUILD_TYPE}/")
    link_directories("d:/work/deep_server/3rd/actor-framework/build/lib/${CMAKE_BUILD_TYPE}/")

if(1)
    include_directories("c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/include/")
    target_link_libraries(${PROJECT_NAME}
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/cublas.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/cublas_device.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/cuda.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/cudadevrt.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/cudart.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/cufft.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/cufftw.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/curand.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/cusparse.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/nppc.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/nppi.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/npps.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/nvblas.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/nvcuvid.lib"
        "c:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v6.5/lib/x64/OpenCL.lib"
        )
endif()

    target_link_libraries(${PROJECT_NAME}
        "d:/work/deep_server/3rd/actor-framework/build64/lib/Debug/caf_core_static.lib"
        "d:/work/deep_server/3rd/actor-framework/build64/lib/Debug/caf_io_static.lib"
        "d:/work/caffe-windows/caffe/build/lib/Debug/caffe-d.lib"
        "d:/work/deep_server/3rd/jsoncpp-master/build/src/lib_json/Debug/jsoncpp.lib"
        "d:/work/yolo_centos/lib/debug/dnl.lib"
        "d:/work/yolo_centos/3rdparty/lib/x64/pthreadVC2.lib"
        ws2_32
        kernel32.lib
        user32.lib
        gdi32.lib
        winspool.lib
        shell32.lib
        ole32.lib
        oleaut32.lib
        uuid.lib
        comdlg32.lib
        advapi32.lib
        ws2_32.lib
        iphlpapi.lib
        d:/work/caffe-windows/caffe/build/lib/Debug/caffe-d.lib
        d:/work/caffe-windows/caffe/build/lib/Debug/proto-d.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/boost_system-vc140-mt-gd-1_61.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/boost_thread-vc140-mt-gd-1_61.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/boost_filesystem-vc140-mt-gd-1_61.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/boost_chrono-vc140-mt-gd-1_61.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/boost_date_time-vc140-mt-gd-1_61.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/boost_atomic-vc140-mt-gd-1_61.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/glogd.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/Lib/gflagsd.lib
        shlwapi.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/libprotobufd.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/caffehdf5_hl_D.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/caffehdf5_D.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/cmake/../lib/caffezlibd.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/lmdbd.lib
        ntdll.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/leveldbd.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/cmake/../lib/boost_date_time-vc140-mt-gd-1_61.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/cmake/../lib/boost_filesystem-vc140-mt-gd-1_61.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/cmake/../lib/boost_system-vc140-mt-gd-1_61.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/snappy_staticd.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/caffezlibd.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/x64/vc14/lib/opencv_highgui310d.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/x64/vc14/lib/opencv_videoio310d.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/x64/vc14/lib/opencv_imgcodecs310d.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/x64/vc14/lib/opencv_imgproc310d.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/x64/vc14/lib/opencv_core310d.lib
        C:/Users/Administrator/.caffe/dependencies/libraries_v140_x64_py27_1.1.0/libraries/lib/libopenblas.dll.a
        )

    #tensorflow related
    include_directories(
        "${TENSORFLOW_FOLDER}/tensorflow/contrib/cmake/build"
        "${TENSORFLOW_FOLDER}/tensorflow/contrib/cmake/build/external/eigen_archive"
        "${TENSORFLOW_FOLDER}/tensorflow/contrib/cmake/build/external/jpeg_archive"
        "${TENSORFLOW_FOLDER}/tensorflow/contrib/cmake/build/external/protobuf/src/"

        )

    target_link_libraries(${PROJECT_NAME} 
        ${TENSORFLOW_FOLDER}/tensorflow/contrib/cmake/build/Release/libtensorflow.lib
        ${TENSORFLOW_FOLDER}/tensorflow/contrib/cmake/build/Release/tf_protos_cc.lib
        ${TENSORFLOW_FOLDER}/tensorflow/contrib/cmake/build/protobuf/src/protobuf/Release/libprotoc.lib
        ${TENSORFLOW_FOLDER}/tensorflow/contrib/cmake/build/protobuf/src/protobuf/Release/libprotobuf-lite.lib
        ${TENSORFLOW_FOLDER}/tensorflow/contrib/cmake/build/protobuf/src/protobuf/Release/libprotobuf.lib
        ) 


else()
    set(CAFFE_PATH "${PROJECT_SOURCE_DIR}/3rd/auto_caffe")
    include_directories("${CAFFE_PATH}/include/")
    include_directories("${CAFFE_PATH}/build/include/")
    include_directories("${PROJECT_SOURCE_DIR}/3rd/actor-framework/libcaf_io/")
    include_directories("${PROJECT_SOURCE_DIR}/3rd/actor-framework/libcaf_core/")
    include_directories("${PROJECT_SOURCE_DIR}/3rd/jsoncpp/include/")
    include_directories("${PROJECT_SOURCE_DIR}/3rd/auto_yolo/src/")
    include_directories (
        /usr/local/include
        /usr/include
        /usr/include/openblas
        /usr/include/python2.7
        /usr/local/cuda/include
        )
    link_directories("${PROJECT_SOURCE_DIR}/3rd/jsoncpp-master/build/src/lib_json/")
    link_directories("${PROJECT_SOURCE_DIR}/3rd/actor-framework/build/lib/")
    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} 
        "/usr/local/lib"
        "/usr/local/lib64")

    target_link_libraries(${PROJECT_NAME}
        "${PROJECT_SOURCE_DIR}/3rd/actor-framework/build/lib/libcaf_core.so"
        "${PROJECT_SOURCE_DIR}/3rd/actor-framework/build/lib/libcaf_io.so"
        "${CAFFE_PATH}/build/lib/libcaffe.so"
        "${PROJECT_SOURCE_DIR}/3rd/jsoncpp/build/src/lib_json/libjsoncpp.a"
        "${PROJECT_SOURCE_DIR}/3rd/auto_yolo/lib/libdnl.so"
        #/lib64/libgflags.so
        #/usr/lib64/libglog.so
        )

    #tensorflow related
    include_directories(
        "${TENSORFLOW_FOLDER}/output/output/external/eigen_archive"
        "${TENSORFLOW_FOLDER}/output/output/external/protobuf/src/"
        "${TENSORFLOW_FOLDER}/output/output/execroot/tensorflow/bazel-out/local_linux-opt/genfiles/external/org_tensorflow"
        "${TENSORFLOW_FOLDER}/output/output/execroot/tensorflow/bazel-out/local-opt/genfiles"
        "${TENSORFLOW_FOLDER}/output/output/execroot/tensorflow/bazel-out/local_linux-opt/genfiles"

        )

    target_link_libraries(${PROJECT_NAME} 
        ${TENSORFLOW_FOLDER}/bazel-bin/tensorflow/libtensorflow_all.so
        "${TENSORFLOW_FOLDER}/output/output/execroot/tensorflow/bazel-out/local_linux-opt/bin/external/jpeg/libsimd_x86_64.a"
        "${TENSORFLOW_FOLDER}/output/output/execroot/tensorflow/bazel-out/local_linux-opt/bin/external/jpeg/libjpeg.a"
        ) 
    
endif()


# boost support
set(Boost_USE_MULTITHREADED ON)
find_package(Boost COMPONENTS system filesystem date_time)
if(Boost_FOUND)
    include_directories(${Boost_INCLUDE_DIRS})
    target_link_libraries(${PROJECT_NAME} ${Boost_LIBRARIES})
endif() 

find_package(OpenCV QUIET COMPONENTS core highgui imgproc imgcodecs)
if(NOT OpenCV_FOUND) # if not OpenCV 3.x, then imgcodecs are not found
    find_package(OpenCV REQUIRED COMPONENTS core highgui imgproc)
endif()
if(OpenCV_FOUND)
    include_directories(${OpenCV_INCLUDE_DIRS})
    target_link_libraries(${PROJECT_NAME} ${OpenCV_LIBS})
endif() 

include(cmake/ProtoBuf.cmake)

set(proto_gen_folder "${PROJECT_BINARY_DIR}/include/proto")
file(GLOB proto_files src/proto/*.proto)
deep_protobuf_generate_cpp_py(${proto_gen_folder} proto_srcs proto_hdrs proto_python ${proto_files})

# include python files either to force generation
add_library(deep_proto STATIC ${proto_hdrs} ${proto_srcs} ${proto_python})
#deep_default_properties(deep_proto)
target_link_libraries(deep_proto PUBLIC ${PROTOBUF_LIBRARIES})
target_include_directories(deep_proto PUBLIC ${PROTOBUF_INCLUDE_DIR})

#list(INSERT Caffe_LINKER_LIBS 0 PUBLIC deep_proto) # note, crucial to prepend!
include_directories(${PROTOBUF_INCLUDE_DIR} "${PROJECT_BINARY_DIR}/include/proto")
target_link_libraries(${PROJECT_NAME} 
    PUBLIC deep_proto)

if(MSVC)
    #execute_process(COMMAND ${CMAKE_COMMAND} -E copy ${proto_python} "d:/work/python/opencv/deep_pb2.py")
    configure_file(${proto_python} "d:/work/python/opencv/deep_pb2.py" COPYONLY)
    configure_file(${proto_python} "d:/work/python/deep_learning/deep_pb2.py" COPYONLY)
endif()

if(MSVC)
    set_target_properties(${PROJECT_NAME} PROPERTIES LINK_FLAGS "/ignore:4819")
    add_definitions( "/W3 /D_CRT_SECURE_NO_WARNINGS /wd4005 /wd4819 /wd4996 /nologo" )
endif()


# ---[ Configuration summary
deep_print_configuration_summary()

