cmake_minimum_required(VERSION 2.8)

set(appname kfusion)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -O3                  -Wall -Wno-unknown-pragmas -Wextra -Wno-error=deprecated-declarations -Wno-error=array-bounds")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -O3  --std=c++11 -Wall -Wno-unknown-pragmas -Wextra -Wno-error=deprecated-declarations -Wno-error=array-bounds ")

IF(ANDROID)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}   -Wno-error ")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-error  ")
ENDIF()

# ------ Declare generate_slam_library function ------
SET(SLAMBENCH_C_WRAPPER	-Wl,--whole-archive -L${WRAPPER_PATH} slambench-c-wrapper	-Wl,--no-whole-archive)
SET(NULL_FILE_PATH	${WRAPPER_PATH}/null.cpp)

function(generate_slam_library library_name )

    add_library(${library_name}-library SHARED ${NULL_FILE_PATH})
    target_link_libraries(${library_name}-library PRIVATE ${SLAMBENCH_C_WRAPPER} ${library_name})
    SET_TARGET_PROPERTIES(${library_name}-library PROPERTIES LINK_FLAGS "${ARGN}")
    install(TARGETS ${library_name}-library  DESTINATION  lib/)

endfunction()



include_directories(${EIGEN3_INCLUDE_DIR})
include_directories(${TOON_INCLUDE_PATHS})


if(APPLE)
  set(common_libraries     stdc++ pthread)
else(APPLE)
  if(ANDROID)
    set(common_libraries      stdc++  android log)
  else (ANDROID)
    set(common_libraries     rt stdc++ pthread)
  endif(ANDROID)
endif(APPLE)




include_directories(${SLAMBENCH_INCLUDE_DIR})
include_directories(./)
   
 # ----------------- CPP VERSION ----------------- 

add_library(${appname}-cpp kernels.cpp kfusion_library.cpp)
target_link_libraries(${appname}-cpp   ${common_libraries})	
SET_TARGET_PROPERTIES(${appname}-cpp PROPERTIES COMPILE_FLAGS "-fPIC")
install(TARGETS ${appname}-cpp  DESTINATION  lib/)   

 # ----------------- OPENMP VERSION ----------------- 

add_library(${appname}-openmp  kernels.cpp kfusion_library.cpp)
target_link_libraries(${appname}-openmp   ${common_libraries})	
SET_TARGET_PROPERTIES(${appname}-openmp PROPERTIES COMPILE_FLAGS "-fPIC -fopenmp")
install(TARGETS ${appname}-openmp  DESTINATION  lib/)


 # ----------------- DYNAMIC  ----------------- 
 
target_include_directories(${appname}-cpp  PRIVATE  ./cpp_headers )
target_include_directories(${appname}-openmp  PRIVATE  ./cpp_headers )
generate_slam_library(${appname}-cpp "")


 #  ----------------- OCL VERSION ----------------- 
 
find_package(OpenCL)
   
if (OPENCL_FOUND) 
   include_directories(${OPENCL_INCLUDE_DIRS})
    add_library(${appname}-opencl  kernels_ocl.cpp common_opencl.cpp  kfusion_library.cpp)
    target_include_directories(${appname}-opencl  PRIVATE  ./cpp_headers )
    target_link_libraries(${appname}-opencl   ${common_libraries} ${OPENCL_LIBRARIES})	
    SET_TARGET_PROPERTIES(${appname}-opencl PROPERTIES COMPILE_FLAGS "-fPIC")
    
    install(TARGETS ${appname}-opencl  DESTINATION  lib/)
 

    generate_slam_library(${appname}-opencl "")

ENDIF()


 #  ----------------- CUDA VERSION ----------------- 

find_package(CUDA)
STRING(REPLACE "_static.a" ".so" CUDA_LIBRARIES "${CUDA_LIBRARIES}")


if (CUDA_FOUND) 

    SET( CUDA_PROPAGATE_HOST_FLAGS OFF ) 

    set(kfusion_cuda_srcs kernels.cu)
	    	
	cuda_add_library(${appname}-cuda    SHARED  ${kfusion_cuda_srcs}  kfusion_library.cpp OPTIONS -O3 -std=c++11 -Xcompiler -fPIC -D_MWAITXINTRIN_H_INCLUDED -D_FORCE_INLINES -I${CMAKE_CURRENT_SOURCE_DIR}/cuda_headers )
	target_link_libraries(${appname}-cuda   ${common_libraries})	
	target_include_directories(${appname}-cuda  PUBLIC ./cuda_headers )
    	install(TARGETS ${appname}-cuda  DESTINATION  lib/)

    
	generate_slam_library(${appname}-cuda "")
	
endif(CUDA_FOUND)




