# 在代码中使用gdb的print变量功能的一种方式

**代码如下：**

```c
#include <stdio.h> //printf
 #include <stdlib.h> //calloc, system
 extern const char *__progname;
 struct person
 {
     int age; 
     int height; 
 };
 static struct person *johndoe;
 static char report[255]; 
 static void printout_struct(void* invar, char* structname){
     /* dumpstack(void) Got this routine from http://www.whitefang.com/unix/faq_toc.html
     ** Section 6.5. Modified to redirect to file to prevent clutter
     */
     /* This needs to be changed... */
     char dbx[160];
     sprintf(dbx, "echo 'p (struct %s)*%p\n' > gdbcmds", structname, invar );
     system(dbx);
     sprintf(dbx, "echo 'where\ndetach' | gdb -batch --command=gdbcmds %s %d > struct.dump", __progname, getpid() );
     system(dbx);
     sprintf(dbx, "cat struct.dump");
     system(dbx);
     return;
 }
 int main ()
 {
     johndoe = (struct person *)calloc(1, sizeof(struct person));
     johndoe->age = 6; 
     printout_struct(johndoe, "person"); 
     johndoe->age = 8; 
     printout_struct(johndoe, "person"); 
     printf("Hello World - age: %d\n:", johndoe->age);
     free(johndoe);
 }
```

```bash
gcc -g -O0 $FN.c -o $FN
```

基本上最终显示了我想要的内容：

```bash
0x00740422 in __kernel_vsyscall ()
 $1 = {age = 6, height = 0}
 0x00740422 in __kernel_vsyscall ()
 $1 = {age = 8, height = 0}
 Hello World - age: 8
```

<https://cpp.hotexamples.com/examples/-/-/PROXY_TRACE/cpp-proxy_trace-function-examples.html>
<https://www.cnblogs.com/feihongwuhen/archive/2010/05/17/7170469.html>
<https://gist.github.com/quark-zju/a20ae638601d2fac476e>
<http://www.verysource.com/code/24731178_1/lua-tests-wrapper.sh.in.html>

```none
# PROXY_TRACE="gdb --batch --command=/Users/jan/projects/in-bzr/mysql-lb/backtrace.gdb --args " 
```

原文链接：[vLinux C: Easy & 'pretty' dump/printout of structs (like in gdb) - from source code?](https://www.faqcode4u.com/faq/73091/linux-c-easy-pretty-dump-printout-of-structs-like-in-gdb-from-source-co)

我正在构建的内核模块中的一些结构有一点问题，所以我认为如果有一种简单的方法可以打印出结构及其值会很好 - 下面是我的意思的一个小的用户空间示例

假设我们有如下简单的 C 示例（以 bash 命令的形式给出）：

```c
FN=mtest
 cat > $FN.c <<EOF
 #include <stdio.h> //printf
 #include <stdlib.h> //calloc
 struct person
 {
  int age; 
  int height; 
 };
 static struct person *johndoe;
 main ()
 {
  johndoe = (struct person *)calloc(1, sizeof(struct person));
  johndoe->age = 6; 
  asm("int3"); //breakpoint for gdb
  printf("Hello World - age: %d\n", johndoe->age);
  free(johndoe);
 }
 EOF
 gcc -g -O0 $FN.c -o $FN
 # just a run command for gdb
 cat > ./gdbcmds <<EOF
 run
 EOF
 gdb --command=./gdbcmds ./$FN
```

如果我们运行此示例，则该程序将编译，而GDB将运行它，并在断点上自动停止。 在这里，我们可以做以下操作：

```c
Program received signal SIGTRAP, Trace/breakpoint trap.
 main () at mtest.c:20
 20  printf("Hello World - age: %d\n", johndoe->age);
 (gdb) p johndoe
 $1 = (struct person *) 0x804b008
 (gdb) p (struct person)*0x804b008
 $2 = {age = 6, height = 0}
 (gdb) c
 Continuing.
 Hello World - age: 6
 Program exited with code 0300.
 (gdb) q
```

如图所示，在GDB中，我们可以打印出（转储？）struct指针Johndoe的值{age = 6，高度= 0} ...我想做同样的事情，但直接来自C程序。 如以下示例所述：

```c
#include <stdio.h> //printf
 #include <stdlib.h> //calloc
 #include <whatever.h> //for imaginary printout_struct
 struct person
 {
  int age; 
  int height; 
 };
 static struct person *johndoe;
 static char report[255]; 
 main ()
 {
  johndoe = (struct person *)calloc(1, sizeof(struct person));
  johndoe->age = 6; 
  printout_struct(johndoe, report); //imaginary command
  printf("Hello World - age: %d\nreport: %s", johndoe->age, report);
  free(johndoe);
 }
```

这将导致如下输出：

```shellsession
Hello World - age: 6
 $2 = {age = 6, height = 0}
```

所以我的问题是——是否存在像想象中的 printout\_struct 这样的函数——或者是否有另一种方法可以使这样的打印输出成为可能？ 提前感谢您的帮助，干杯！

## 回答1

只是想说-感谢您所有出色且令人难以置信的快速答案，帮助我理解了这个问题（为什么C中没有这样的“本机”函数）！ （很抱歉回答我自己的问题 - 这样做，以免混淆原始帖子，并能够格式化代码）在进一步寻找时，我设法找到：

* [generate a core dump in linux - Stack Overflow](https://stackoverflow.com/questions/17965/generate-a-core-dump-in-linux)
* [just-in-time debugging? - mlist.linux.kernel | Google Groups](https://groups.google.com/group/mlist.linux.kernel/msg/7f8a9393cde343d1)

这说明了使用进程本身的 pid 调用 gdb 的技巧，因此我修改了那里找到的 dumpstack 函数，以获得以下代码：

```c
FN=mtest
 cat > $FN.c <<EOF
 #include <stdio.h> //printf
 #include <stdlib.h> //calloc, system
 extern const char *__progname;
 struct person
 {
     int age; 
     int height; 
 };
 static struct person *johndoe;
 static char report[255]; 
 static void printout_struct(void* invar, char* structname){
     /* dumpstack(void) Got this routine from http://www.whitefang.com/unix/faq_toc.html
     ** Section 6.5. Modified to redirect to file to prevent clutter
     */
     /* This needs to be changed... */
     char dbx[160];
     sprintf(dbx, "echo 'p (struct %s)*%p\n' > gdbcmds", structname, invar );
     system(dbx);
     sprintf(dbx, "echo 'where\ndetach' | gdb -batch --command=gdbcmds %s %d > struct.dump", __progname, getpid() );
     system(dbx);
     sprintf(dbx, "cat struct.dump");
     system(dbx);
     return;
 }
 main ()
 {
     johndoe = (struct person *)calloc(1, sizeof(struct person));
     johndoe->age = 6; 
     printout_struct(johndoe, "person"); 
     johndoe->age = 8; 
     printout_struct(johndoe, "person"); 
     printf("Hello World - age: %d\n:", johndoe->age);
     free(johndoe);
 }
 EOF
 gcc -g -O0 $FN.c -o $FN
 ./$FN
```

基本上最终显示了我想要的内容：

```
0x00740422 in __kernel_vsyscall ()
 $1 = {age = 6, height = 0}
 0x00740422 in __kernel_vsyscall ()
 $1 = {age = 8, height = 0}
 Hello World - age: 8
```

虽然，我不确定它是否适用于内核模块......再次感谢您的帮助，干杯！

**编辑**：我认为它不适用于内核模块的原因是，在这种情况下，我们有一个带有进程 ID 的用户态程序； 我们只是从这个程序中调用 gdb，同时告诉它我们的 PID - 所以 gdb 可以“附加”到我们的进程； 然后，由于 gdb 还被指示加载带有调试符号的可执行文件（因此它将“知道”结构是什么），并指示给定结构变量所在的地址，然后 gdb 可以打印输出结构。

对于内核模块 - 首先我不认为它们是具有唯一 PID 意义上的“进程”，因此 gdb 将没有任何附加内容！ 事实上，有一个内核调试器，kgdb，它实际上可以闯入一个正在运行的内核并单步调试模块源代码； 但是，您需要通过串行连接连接第二台机器 - 或虚拟机，请参阅 [Linux Hacks: Setting up kgdb using kvm/qemu](https://linux-hacks.blogspot.com/2008/05/setting-up-kgdb-using-kvmqemu.html).

## Some Code Answers

```c
FN=mtest  cat >
$FN.c <<EOF #include <stdio.h>
//printf #include <stdlib.h>
//calloc  struct person {  int age;
  int height;
 };
 static struct person *johndoe;
 main () {   johndoe = (struct person *)calloc(1, sizeof(struct person));
 johndoe->age = 6;
   asm("int3");
//breakpoint for gdb   printf("Hello World - age: %d\n", johndoe->age);
  free(johndoe);
} EOF  gcc -g -O0 $FN.c -o $FN  # just a run command for gdb cat >
./gdbcmds <<EOF run EOF  gdb --command=./gdbcmds ./$FN
```

---

```bash
Program received signal SIGTRAP, Trace/breakpoint trap. main () at mtest.c:20 20  printf("Hello World - age: %d\n", johndoe->age);
(gdb) p johndoe $1 = (struct person *) 0x804b008 (gdb) p (struct person)*0x804b008 $2 = {age = 6, height = 0} (gdb) c Continuing. Hello World - age: 6  Program exited with code 0300. (gdb) q
```

---

```c
#include <stdio.h>
//printf #include <stdlib.h>
//calloc #include <whatever.h>
//for imaginary printout_struct  struct person {  int age;
  int height;
 };
 static struct person *johndoe;
static char report[255];
  main () {   johndoe = (struct person *)calloc(1, sizeof(struct person));
 johndoe->age = 6;
   printout_struct(johndoe, report);
//imaginary command   printf("Hello World - age: %d\nreport: %s", johndoe->age, report);
  free(johndoe);
}
```

---

```bash
Hello World - age: 6 $2 = {age = 6, height = 0}
```

---

```c
FN=mtest  cat >
$FN.c <<EOF #include <stdio.h>
//printf #include <stdlib.h>
//calloc, system  extern const char *__progname;
 struct person {
int age;
int height;
 };
 static struct person *johndoe;
static char report[255];
  static void printout_struct(void* invar, char* structname){
/* dumpstack(void) Got this routine from http://www.whitefang.com/unix/faq_toc.html
** Section 6.5. Modified to redirect to file to prevent clutter
*/
/* This needs to be changed... */
char dbx[160];
sprintf(dbx, "echo 'p (struct %s)*%p\n' >
gdbcmds", structname, invar );
system(dbx);
sprintf(dbx, "echo 'where\ndetach' | gdb -batch --command=gdbcmds %s %d >
struct.dump", __progname, getpid() );
system(dbx);
sprintf(dbx, "cat struct.dump");
system(dbx);
return;
}  main () {
johndoe = (struct person *)calloc(1, sizeof(struct person));
johndoe->age = 6;
printout_struct(johndoe, "person");
johndoe->age = 8;
printout_struct(johndoe, "person");
printf("Hello World - age: %d\n:", johndoe->age);
free(johndoe);
}   EOF  gcc -g -O0 $FN.c -o $FN  ./$FN
```

---

```bash
0x00740422 in __kernel_vsyscall () $1 = {age = 6, height = 0} 0x00740422 in __kernel_vsyscall () $1 = {age = 8, height = 0} Hello World - age: 8
```

---

```c
#define gdb_print(v)  gdb_print(huge_struct);
  gdb-print-prepare() {
# usage:
# gdb-print-prepare $src >
app.gdb
# gdb --batch --quiet --command=app.gdb app
cat  <<-EOF
set auto-load safe-path /
EOF
grep --with-filename --line-number --recursive '^\s\+gdb_print(.*);' $1 | \
while IFS=$'\t ;()' read line func var rest;
do
  cat  <<-EOF
  break ${line%:}
  commands
  silent
  where 1
  echo \\n$var\\n
  print $var
  cont
  end
  EOF
done
cat  <<-EOF
run
bt
echo ---\\n
EOF }
```
