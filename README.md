# miniKV
MiniKV-C is a Redis-inspired in-memory key-value store implemented in C.

The project is designed to explore low-level systems programming concepts such as dynamic memory allocation, pointer ownership, hash tables, collision handling, resizing and rehashing, command parsing, file-based persistence, and memory debugging with Valgrind.

The database supports simple commands such as `SET`, `GET`, `DEL`, `EXISTS`, `SAVE`, and `LOAD`. Later versions may include a TCP server mode and a small client interface.

## Goals

- Implement a custom hash table in C
- Handle collisions using separate chaining
- Support dynamic resizing and rehashing
- Build a command-line interface
- Add save/load persistence
- Write unit tests
- Ensure the project is Valgrind-clean
