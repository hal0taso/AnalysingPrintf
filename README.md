# [check]選-A-4.
C言語のprintf()関数またはUNIXのfork()というシステムコールについて、これらはどのようなものですか？　数値や文字列を表示する・プロセスを作るというだけではなく、深堀りして考え、疑問を持ち、手を動かして調べてわかったことを教えてください。

### 方針
- printfの実行をgdbを使って関数の内部に進んで行く
	- どういう関数が呼ばれるのか

---
printf()関数について調べたことを述べます。実行環境は以下のとおりです。

```
$ uname -a
Linux ubuntu-vm 3.19.0-80-generic #88~14.04.1-Ubuntu SMP Fri Jan 13 14:54:07 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
$ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 14.04.5 LTS
Release:	14.04
Codename:	trusty
$ gcc --version
gcc (Ubuntu 4.8.4-2ubuntu1~14.04.3) 4.8.4
Copyright (C) 2013 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

一般的に、printf文は次のような形式で書けます。

```
printf(const char *format, ...); 
```

まず、以下のようなプログラムを作成し、コンパイルして実行します。

```
$ cat helloworld.c
#include <stdio.h>

int main(int argc, char *argv[]){
  printf("Hello, World!\n");
  return 0;
}
$ gcc -m32 helloworld.c -o helloworld -O0 -g -Wall
$ ./helloworld 
Hello, World!
```

main()関数の逆アセンブル結果をobjdumpで確認します。

```
$ objdump -M intel -d helloworld | grep '<main>' -A13
0804841d <main>:
 804841d:	55                   	push   ebp
 804841e:	89 e5                	mov    ebp,esp
 8048420:	83 e4 f0             	and    esp,0xfffffff0
 8048423:	83 ec 10             	sub    esp,0x10
 8048426:	c7 04 24 d0 84 04 08 	mov    DWORD PTR [esp],0x80484d0
 804842d:	e8 be fe ff ff       	call   80482f0 <puts@plt>
 8048432:	b8 00 00 00 00       	mov    eax,0x0
 8048437:	c9                   	leave  
 8048438:	c3                   	ret    
 8048439:	66 90                	xchg   ax,ax
 804843b:	66 90                	xchg   ax,ax
 804843d:	66 90                	xchg   ax,ax
 804843f:	90                   	nop
```

これを見て気になったのは、ソースコードではprintfを使って文字列の表示を行ったにも関わらず0x804842dで呼ばれているのはputs()関数であるという点です。

そこで、どのような関数が呼ばれているのか、つまりprintf()が使われていないことを確かめるためにreadelfコマンドでシンボルテーブルを確認しました。

```
$ readelf -r helloworld

Relocation section '.rel.dyn' at offset 0x290 contains 1 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
08049ffc  00000206 R_386_GLOB_DAT    00000000   __gmon_start__

Relocation section '.rel.plt' at offset 0x298 contains 3 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
0804a00c  00000107 R_386_JUMP_SLOT   00000000   puts
0804a010  00000207 R_386_JUMP_SLOT   00000000   __gmon_start__
0804a014  00000307 R_386_JUMP_SLOT   00000000   __libc_start_main
```

やはりprintf()関数は使われず、puts()関数が使われているようです。これはコンパイラの最適化によって文字列の末尾が"\n"で終わり、フォーマット指定子が存在しない時、printf()はputs()に置き換えられることが原因です。しかし、今回は最適化を無効にしたはずなので、puts()が呼ばれていたことに驚きました。試しに、次のようなhelloworld1.cというプログラムをコンパイルしてシンボルテーブルを確認してみます。

```
#include <stdio.h>

int main(int argc, char *argv[]){
  printf("Hello, World!");
  return 0;
}
```

結果はこのようになりました。

```
$ gcc -m32 helloworld1.c -o helloworld1 -g -Wall
ubuntu@ubuntu-vm:~/Projects/AnalysingPrintf$ readelf helloworld1 -r

Relocation section '.rel.dyn' at offset 0x294 contains 1 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
08049ffc  00000206 R_386_GLOB_DAT    00000000   __gmon_start__

Relocation section '.rel.plt' at offset 0x29c contains 3 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
0804a00c  00000107 R_386_JUMP_SLOT   00000000   printf
0804a010  00000207 R_386_JUMP_SLOT   00000000   __gmon_start__
0804a014  00000307 R_386_JUMP_SLOT   00000000   __libc_start_main
```

確かにprintfが使われていることがわかります。最適化を無効にしてもgcc-4.8.8以降はputs()に置き換えられるようです。

次に、フォーマット指定子を用いて、以下のようなプログラムhelloargv.cを実行してみます。

```
$ cat helloargv0.c 
#include <stdio.h>

int main(int argc, char *argv[]){
  printf("Hello, %s\n", argv[0]);
  return 0;
}
$ gcc -m32 helloargv0.c -o helloargv0 -O0 -g -Wall
ubuntu@ubuntu-vm:~/Projects/AnalysingPrintf$ ./helloargv0 
Hello, ./helloargv0
```

ところで、気になったのは、printfはいつ標準入出力に文字列を表示しているのか？ということです。

そこで、helloargv0をgdbでデバッグしてみます。
