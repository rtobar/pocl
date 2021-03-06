#=============================================================================
#   CMake build system files
#
#   Copyright (c) 2014 pocl developers
#
#   Permission is hereby granted, free of charge, to any person obtaining a copy
#   of this software and associated documentation files (the "Software"), to deal
#   in the Software without restriction, including without limitation the rights
#   to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
#   copies of the Software, and to permit persons to whom the Software is
#   furnished to do so, subject to the following conditions:
#
#   The above copyright notice and this permission notice shall be included in
#   all copies or substantial portions of the Software.
#
#   THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
#   IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
#   FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
#   AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
#   LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
#   OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
#   THE SOFTWARE.
#
#=============================================================================

set(LLVMPASSES_SOURCES "Barrier.h"
  "Kernel.h" "Kernel.cc"
  "ParallelRegion.h" "ParallelRegion.cc"
  "CanonicalizeBarriers.h" "CanonicalizeBarriers.cc"
  "LoopBarriers.h" "LoopBarriers.cc"
  "Workgroup.h" "Workgroup.cc"
  "BarrierTailReplication.h" "BarrierTailReplication.cc"
  "Flatten.cc" "FlattenGlobals.cc"
  "IsolateRegions.h" "IsolateRegions.cc"
  "WorkitemReplication.h" "WorkitemReplication.cc"
  "ImplicitLoopBarriers.h" "ImplicitLoopBarriers.cc"
  "WorkItemAliasAnalysis.cc"
  "WorkitemHandler.h" "WorkitemHandler.cc"
  "WorkitemLoops.h" "WorkitemLoops.cc"
  "PHIsToAllocas.h" "PHIsToAllocas.cc"
  "BreakConstantGEPs.h" "BreakConstantGEPs.cpp"
  "WorkitemHandlerChooser.h" "WorkitemHandlerChooser.cc"
  "AllocasToEntry.h" "AllocasToEntry.cc"
  "LLVMUtils.cc" "LLVMUtils.h" "LLVMFileUtils.cc"
  "VariableUniformityAnalysis.h" "VariableUniformityAnalysis.cc"
  "AutomaticLocals.cc" "ImplicitConditionalBarriers.cc"
  "ImplicitConditionalBarriers.h"
  "DebugHelpers.h" "DebugHelpers.cc"
  "RemoveBarrierCalls.h" "RemoveBarrierCalls.cc"
  "HandleSamplerInitialization.h" "HandleSamplerInitialization.cc"
  "RemoveOptnoneFromWIFunc.h" "RemoveOptnoneFromWIFunc.cc"
  "OptimizeWorkItemFuncCalls.h" "OptimizeWorkItemFuncCalls.cc")

if(POCL_USE_FAKE_ADDR_SPACE_IDS)
list(APPEND LLVMPASSES_SOURCES "TargetAddressSpaces.cc")
endif()

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${LLVM_CFLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${LLVM_CXXFLAGS}")

include_directories(${LLVM_INCLUDE_DIRS})
include_directories(../CL)
include_directories(../CL/devices)

if(MSVC)
  set_source_files_properties( ${LLVMPASSES_SOURCES} PROPERTIES LANGUAGE CXX )
endif(MSVC)

# This library (just a bunch of objects really) is linked against libpocl
# to call the kernel compiler passes directly from the host library.
#noinst_LTLIBRARIES = libllvmpasses.la
add_library("llvmpasses" OBJECT ${LLVMPASSES_SOURCES} "linker.cpp" "linker.h")

# This creates a separate llvmopencl.so that is loaded to 'opt'
# for the pocl-workgroup version.
#pkglib_LTLIBRARIES = llvmopencl.la
add_library("llvmopencl" MODULE "$<TARGET_OBJECTS:llvmpasses>")

# not sure where other platforms get their library linkage list, probably there is
# way to make this more consistent on different platforms
if(MSVC)
  target_link_libraries("llvmopencl" ${POCL_LLVM_LIBS})
endif(MSVC)

#target_link_libraries("llvmopencl" "${LLVM_SYSLIBS}")
 
# TODO cmake should handle this, i might me missing something …
if(APPLE)
  set(ADD_LD_FLAGS "-Wl,-undefined -Wl,dynamic_lookup ")
else()
  set(ADD_LD_FLAGS "-Wl,-export-dynamic")
endif()

# TODO ENABLE_EXPORTS -> Wl,export_dynamic
# http://www.cmake.org/cmake/help/v2.8.12/cmake.html#prop_tgt:ENABLE_EXPORTS
# http://www.cmake.org/pipermail/cmake/2011-November/047708.html
set_target_properties("llvmopencl" PROPERTIES LINK_FLAGS "${LLVM_LDFLAGS} ${ADD_LD_FLAGS}")
if(NOT APPLE)
  set_target_properties("llvmopencl" PROPERTIES SOVERSION "${KERNEL_COMPILER_LIB_VERSION}" VERSION "${KERNEL_COMPILER_LIB_VERSION}")
endif()

install(TARGETS "llvmopencl"
        LIBRARY DESTINATION ${POCL_INSTALL_PRIVATE_LIBDIR}
        ARCHIVE DESTINATION ${POCL_INSTALL_PRIVATE_LIBDIR}/static)
