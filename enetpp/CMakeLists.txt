cmake_minimum_required(VERSION 2.8)
project(enetpp)

set(CMAKE_CXX_STANDARD 14)

include_directories(enet/include)
include_directories(enet/include/enet)
include_directories(enet/out)
include_directories(enet/out/include)
include_directories(enet/out/include/enet)
include_directories(include)
include_directories(include/enetpp)

add_executable(enetpp
        enet/include/enet/callbacks.h
        enet/include/enet/enet.h
        enet/include/enet/list.h
        enet/include/enet/protocol.h
        enet/include/enet/time.h
        enet/include/enet/types.h
        enet/include/enet/unix.h
        enet/include/enet/utility.h
        enet/include/enet/win32.h
        enet/out/include/enet/callbacks.h
        enet/out/include/enet/enet.h
        enet/out/include/enet/list.h
        enet/out/include/enet/protocol.h
        enet/out/include/enet/time.h
        enet/out/include/enet/types.h
        enet/out/include/enet/unix.h
        enet/out/include/enet/utility.h
        enet/out/include/enet/win32.h
        enet/callbacks.c
        enet/compress.c
        enet/host.c
        enet/list.c
        enet/packet.c
        enet/peer.c
        enet/protocol.c
        enet/unix.c
        enet/win32.c
        include/enetpp/client.h
        include/enetpp/client_connect_params.h
        include/enetpp/client_queued_packet.h
        include/enetpp/client_statistics.h
        include/enetpp/global_state.h
        include/enetpp/server.h
        include/enetpp/server_event.h
        include/enetpp/server_listen_params.h
        include/enetpp/server_queued_packet.h
        include/enetpp/set_current_thread_name.h
        include/enetpp/trace_handler.h
        sample/main.cpp)
