cmake_minimum_required(VERSION 2.8.11)
project(PSRP_PROVIDER)

# Export commands for auto-completion engines
set(CMAKE_EXPORT_COMPILE_COMMANDS 1)

add_definitions(-D_GNU_SOURCE)

# Cannot use add_compile_options with 2.8.11
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Werror -fPIC")

if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
	add_definitions(-Dmacos)
	set(CMAKE_ICONV iconv)
	set(OUR_LD_PATH export DYLD_LIBRARY_PATH)
else ()
	set(OUR_LD_PATH export LD_LIBRARY_PATH)
endif ()

set(OMI ../omi/Unix)

find_package(Threads REQUIRED)
link_directories(${OMI}/output/lib)

add_library(psrpclient SHARED
	Client.c
	xpress.c
	BufferManipulation.c
	schema.c
	Utilities.c
	)

add_library(psrpomiprov SHARED
	Shell.c
	Command.c
	module.c
	schema.c
	xpress.c
	BufferManipulation.c
	coreclrutil.cpp
	Utilities.c
	)

target_link_libraries(psrpclient
	mi
	base
	pal
	${CMAKE_THREAD_LIBS_INIT}
	${CMAKE_ICONV}
	)

target_link_libraries(psrpomiprov
	mi
	base
	pal
	${CMAKE_THREAD_LIBS_INIT}
	dl
	pam
	ssl
	crypto
	${CMAKE_ICONV})

target_include_directories(psrpomiprov PRIVATE
	${OMI}/output/include ${OMI} ${OMI}/common)

# Cannot use target_compile_options with 2.8.11
set_target_properties(psrpomiprov PROPERTIES COMPILE_FLAGS
	"-fvisibility=hidden -fno-strict-aliasing")

target_include_directories(psrpclient PRIVATE ${OMI}/output/include ${OMI} ${OMI}/common)

set_target_properties(psrpclient PROPERTIES COMPILE_FLAGS
	"-fvisibility=hidden -fno-strict-aliasing")

add_custom_command(TARGET psrpclient POST_BUILD
	COMMAND ${OUR_LD_PATH}=${OMI}/output/lib && ${OMI}/output/bin/chkshlib $<TARGET_FILE:psrpclient>)

add_custom_command(TARGET psrpomiprov POST_BUILD
	COMMAND  ${OUR_LD_PATH}=${OMI}/output/lib && ${OMI}/output/bin/chkshlib $<TARGET_FILE:psrpomiprov>)

add_custom_target(reg
	COMMAND  ${OUR_LD_PATH}=${OMI}/output/lib && ${OMI}/output/bin/omireg $<TARGET_FILE:psrpomiprov> -n interop --hosting @requestor-shell@)

add_custom_target(gen DEPENDS schema.mof
	COMMAND  ${OUR_LD_PATH}=${OMI}/output/lib && ${OMI}/output/bin/omigen -C ${OMI}/share/networkschema/CIM_Schema.mof schema.mof Shell Command)

