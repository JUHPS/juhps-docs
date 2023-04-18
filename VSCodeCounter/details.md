# Details

Date : 2023-04-18 13:43:56

Directory /Users/fengzetao/Workspace/Docker/server/server

Total : 88 files,  11883 codes, 4062 comments, 2589 blanks, all 18534 lines

[Summary](results.md) / Details / [Diff Summary](diff.md) / [Diff Details](diff-details.md)

## Files
| filename | language | code | comment | blank | total |
| :--- | :--- | ---: | ---: | ---: | ---: |
| [Makefile](/Makefile) | Makefile | 25 | 0 | 4 | 29 |
| [README.md](/README.md) | Markdown | 91 | 47 | 68 | 206 |
| [bin/conf/log.yml](/bin/conf/log.yml) | YAML | 17 | 0 | 0 | 17 |
| [bin/conf/server.yml](/bin/conf/server.yml) | YAML | 2 | 0 | 0 | 2 |
| [bin/conf/system.yml](/bin/conf/system.yml) | YAML | 2 | 0 | 0 | 2 |
| [cmake/utils.cmake](/cmake/utils.cmake) | CMake | 30 | 0 | 3 | 33 |
| [generate.sh](/generate.sh) | Shell Script | 31 | 1 | 6 | 38 |
| [src/address.cc](/src/address.cc) | C++ | 459 | 7 | 82 | 548 |
| [src/address.h](/src/address.h) | C++ | 115 | 160 | 49 | 324 |
| [src/application.cc](/src/application.cc) | C++ | 231 | 4 | 42 | 277 |
| [src/application.h](/src/application.h) | C++ | 24 | 1 | 10 | 35 |
| [src/bytearray.cc](/src/bytearray.cc) | C++ | 565 | 0 | 90 | 655 |
| [src/bytearray.h](/src/bytearray.h) | C++ | 91 | 323 | 65 | 479 |
| [src/config.cc](/src/config.cc) | C++ | 118 | 7 | 21 | 146 |
| [src/config.h](/src/config.h) | C++ | 346 | 168 | 52 | 566 |
| [src/daemon.cc](/src/daemon.cc) | C++ | 66 | 2 | 8 | 76 |
| [src/daemon.h](/src/daemon.h) | C++ | 20 | 13 | 9 | 42 |
| [src/endian.h](/src/endian.h) | C++ | 48 | 21 | 15 | 84 |
| [src/env.cc](/src/env.cc) | C++ | 129 | 2 | 21 | 152 |
| [src/env.h](/src/env.h) | C++ | 38 | 70 | 25 | 133 |
| [src/fd_manager.cc](/src/fd_manager.cc) | C++ | 94 | 0 | 15 | 109 |
| [src/fd_manager.h](/src/fd_manager.h) | C++ | 46 | 74 | 19 | 139 |
| [src/fiber.cc](/src/fiber.cc) | C++ | 143 | 33 | 35 | 211 |
| [src/fiber.h](/src/fiber.h) | C++ | 44 | 78 | 22 | 144 |
| [src/hook.cc](/src/hook.cc) | C++ | 416 | 0 | 59 | 475 |
| [src/hook.h](/src/hook.h) | C++ | 59 | 11 | 28 | 98 |
| [src/http/http-parser/http_parser.c](/src/http/http-parser/http_parser.c) | C | 1,886 | 218 | 395 | 2,499 |
| [src/http/http-parser/http_parser.h](/src/http/http-parser/http_parser.h) | C++ | 279 | 105 | 56 | 440 |
| [src/http/http.cc](/src/http/http.cc) | C++ | 312 | 4 | 42 | 358 |
| [src/http/http.h](/src/http/http.h) | C++ | 201 | 417 | 94 | 712 |
| [src/http/http_connection.cc](/src/http/http_connection.cc) | C++ | 357 | 71 | 36 | 464 |
| [src/http/http_connection.h](/src/http/http_connection.h) | C++ | 131 | 184 | 33 | 348 |
| [src/http/http_parser.cc](/src/http/http_parser.cc) | C++ | 253 | 64 | 45 | 362 |
| [src/http/http_parser.h](/src/http/http_parser.h) | C++ | 54 | 101 | 32 | 187 |
| [src/http/http_server.cc](/src/http/http_server.cc) | C++ | 62 | 4 | 14 | 80 |
| [src/http/http_server.h](/src/http/http_server.h) | C++ | 39 | 18 | 18 | 75 |
| [src/http/http_session.cc](/src/http/http_session.cc) | C++ | 48 | 22 | 10 | 80 |
| [src/http/http_session.h](/src/http/http_session.h) | C++ | 16 | 19 | 9 | 44 |
| [src/http/servlet.cc](/src/http/servlet.cc) | C++ | 141 | 0 | 26 | 167 |
| [src/http/servlet.h](/src/http/servlet.h) | C++ | 125 | 103 | 41 | 269 |
| [src/iomanager.cc](/src/iomanager.cc) | C++ | 357 | 55 | 61 | 473 |
| [src/iomanager.h](/src/iomanager.h) | C++ | 55 | 107 | 25 | 187 |
| [src/jujimeizuo.h](/src/jujimeizuo.h) | C++ | 34 | 0 | 1 | 35 |
| [src/library.cc](/src/library.cc) | C++ | 70 | 0 | 11 | 81 |
| [src/library.h](/src/library.h) | C++ | 11 | 0 | 5 | 16 |
| [src/log.cc](/src/log.cc) | C++ | 602 | 67 | 76 | 745 |
| [src/log.h](/src/log.h) | C++ | 191 | 301 | 97 | 589 |
| [src/macro.h](/src/macro.h) | C++ | 30 | 4 | 6 | 40 |
| [src/main.cc](/src/main.cc) | C++ | 13 | 0 | 1 | 14 |
| [src/module.cc](/src/module.cc) | C++ | 141 | 0 | 34 | 175 |
| [src/module.h](/src/module.h) | C++ | 67 | 10 | 23 | 100 |
| [src/mutex.cc](/src/mutex.cc) | C++ | 21 | 0 | 7 | 28 |
| [src/mutex.h](/src/mutex.h) | C++ | 197 | 178 | 54 | 429 |
| [src/noncopyable.h](/src/noncopyable.h) | C++ | 12 | 15 | 8 | 35 |
| [src/scheduler.cc](/src/scheduler.cc) | C++ | 164 | 23 | 32 | 219 |
| [src/scheduler.h](/src/scheduler.h) | C++ | 89 | 75 | 28 | 192 |
| [src/singleton.h](/src/singleton.h) | C++ | 34 | 20 | 12 | 66 |
| [src/socket.cc](/src/socket.cc) | C++ | 421 | 0 | 55 | 476 |
| [src/socket.h](/src/socket.h) | C++ | 92 | 244 | 60 | 396 |
| [src/stream.cc](/src/stream.cc) | C++ | 51 | 0 | 8 | 59 |
| [src/stream.h](/src/stream.h) | C++ | 21 | 81 | 15 | 117 |
| [src/streams/socket_stream.cc](/src/streams/socket_stream.cc) | C++ | 83 | 0 | 15 | 98 |
| [src/streams/socket_stream.h](/src/streams/socket_stream.h) | C++ | 29 | 59 | 16 | 104 |
| [src/tcp_server.cc](/src/tcp_server.cc) | C++ | 112 | 0 | 16 | 128 |
| [src/tcp_server.h](/src/tcp_server.h) | C++ | 129 | 62 | 26 | 217 |
| [src/thread.cc](/src/thread.cc) | C++ | 65 | 0 | 15 | 80 |
| [src/thread.h](/src/thread.h) | C++ | 26 | 39 | 16 | 81 |
| [src/timer.cc](/src/timer.cc) | C++ | 182 | 1 | 25 | 208 |
| [src/timer.h](/src/timer.h) | C++ | 57 | 85 | 19 | 161 |
| [src/uri.cc](/src/uri.cc) | C++ | 102 | 1 | 21 | 124 |
| [src/uri.h](/src/uri.h) | C++ | 42 | 88 | 27 | 157 |
| [src/util.cc](/src/util.cc) | C++ | 492 | 5 | 55 | 552 |
| [src/util.h](/src/util.h) | C++ | 66 | 179 | 42 | 287 |
| [src/worker.cc](/src/worker.cc) | C++ | 93 | 0 | 18 | 111 |
| [src/worker.h](/src/worker.h) | C++ | 66 | 0 | 14 | 80 |
| [template/bin/conf/log.yml](/template/bin/conf/log.yml) | YAML | 17 | 0 | 0 | 17 |
| [template/bin/conf/server.yml](/template/bin/conf/server.yml) | YAML | 2 | 0 | 0 | 2 |
| [template/bin/conf/system.yml](/template/bin/conf/system.yml) | YAML | 2 | 0 | 0 | 2 |
| [template/bin/static/css/btn.css](/template/bin/static/css/btn.css) | CSS | 65 | 0 | 1 | 66 |
| [template/bin/static/css/font-awesome.min.css](/template/bin/static/css/font-awesome.min.css) | CSS | 1 | 3 | 1 | 5 |
| [template/bin/static/css/mask.css](/template/bin/static/css/mask.css) | CSS | 56 | 4 | 1 | 61 |
| [template/bin/static/css/style.css](/template/bin/static/css/style.css) | CSS | 87 | 1 | 7 | 95 |
| [template/bin/static/index.html](/template/bin/static/index.html) | HTML | 61 | 1 | 9 | 71 |
| [template/bin/static/js/boom.js](/template/bin/static/js/boom.js) | JavaScript | 155 | 0 | 8 | 163 |
| [template/bin/static/js/mask.js](/template/bin/static/js/mask.js) | JavaScript | 9 | 1 | 2 | 12 |
| [template/move.sh](/template/move.sh) | Shell Script | 9 | 1 | 2 | 12 |
| [template/template/my_module.cc](/template/template/my_module.cc) | C++ | 36 | 0 | 12 | 48 |
| [template/template/my_module.h](/template/template/my_module.h) | C++ | 12 | 0 | 3 | 15 |

[Summary](results.md) / Details / [Diff Summary](diff.md) / [Diff Details](diff-details.md)