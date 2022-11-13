
```sh
gcc -m32 -fno-asynchronous-unwind-tables -s -O2 -c -o ./c/start.o ./c/start.c
D:\software\objconv\objconv.exe -fnasm ./c/start.o ./c/start.asm

D:\\software\\NASM\\nasm.exe -f win32 .\c\start.asm -o ./c/start.bin
```