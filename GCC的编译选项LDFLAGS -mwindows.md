
# GCC的编译选项LDFLAGS -mwindows #

http://cboard.cprogramming.com/tech-board/82071-mingw-linking-gui-program.html


Compile with `g++ -mwindows hello_win.cpp -o hello_win `

Then run it, and you should get a message box in the centre of the screen. This can also be run from Explorer. Without the -mwindows option, when run from Explorer you would also get a nasty DOS box appear for the duration of the program. However this might be a useful place to send debugging information; if you also include stdio.h you can use printf to display on this window. 