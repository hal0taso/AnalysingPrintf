# [check]選-A-4.
C言語のprintf()関数またはUNIXのfork()というシステムコールについて、これらはどのようなものですか？　数値や文字列を表示する・プロセスを作るというだけではなく、深堀りして考え、疑問を持ち、手を動かして調べてわかったことを教えてください。

### 方針
- printfの実行をgdbを使って関数の内部に進んで行く
	- どういう関数が呼ばれるのか

---
printf関数について調べたことを述べます。実行環境は以下のとおりです。

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
```

一般的に、printf文は次のような形式で書けます。

```
printf(const char *format, ...); 
```

まず、以下のようなプログラムを作成し、コンパイルして実行し、main関数の逆アセンブル結果をobjdumpで見てみました。

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

これを見て気になったのは、ソースコードではprintfを使って文字列の表示を行ったにも関わらず0x804842dで呼ばれているのはputs関数であるという点です.これはコンパイラの最適化によって文字列の末尾が"\n"で終わり、フォーマット指定子が存在しない時、printfはputsに置き換えられることが原因です。しかし、今回は最適化を無効にしたはずなので、putsが呼ばれていたことに驚きました。

次に、フォーマット指定子を用いて以下のようなプログラムhelloargv0.cを実行してみます。

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

ところで、気になったのはprintfはいつ標準入出力に文字列を表示しているのか？ということです。そこで、helloargv0をgdbでデバッグしてみます。

```
$ gdb -q helloargv0
Reading symbols from helloargv0...done.
$ disas main
Dump of assembler code for function main:
   0x0804841d <+0>:	push   ebp
   0x0804841e <+1>:	mov    ebp,esp
   0x08048420 <+3>:	and    esp,0xfffffff0
   0x08048423 <+6>:	sub    esp,0x10
   0x08048426 <+9>:	mov    eax,DWORD PTR [ebp+0xc]
   0x08048429 <+12>:	mov    eax,DWORD PTR [eax]
   0x0804842b <+14>:	mov    DWORD PTR [esp+0x4],eax
   0x0804842f <+18>:	mov    DWORD PTR [esp],0x80484e0
   0x08048436 <+25>:	call   0x80482f0 <printf@plt>
   0x0804843b <+30>:	mov    eax,0x0
   0x08048440 <+35>:	leave  
   0x08048441 <+36>:	ret    
End of assembler dump.
```

アドレス0x8048436でprintf@plt関数が呼ばれているのがわかります。これはprintfのPLT(Procedure Linkage Table)のアドレスです。コンパイル時に共有ライブラリを動的リンクした場合、その関数はGOT(Global Offset Table)と呼ばれるジャンプテーブルを介して呼び出されます。これを確認するために、printf@pltの逆アセンブリ結果を確認します。この中身について調べてみます。

```
$ disas 0x80482f0
Dump of assembler code for function printf@plt:
   0x080482f0 <+0>:	jmp    DWORD PTR ds:0x804a00c
   0x080482f6 <+6>:	push   0x0
   0x080482fb <+11>:	jmp    0x80482e0
End of assembler dump.
```

1行目を見ると、0x804a00cにジャンプしているのがわかります。ライブラリの関数アドレスを保存するGOT領域を探してみると、0x804a00cを見つけることができました。

```
$ readelf helloargv0 -r

Relocation section '.rel.plt' at offset 0x29c contains 3 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
0804a00c  00000107 R_386_JUMP_SLOT   00000000   printf
```

これによって、ライブラリのアドレスを解決して、printfを実行しています。実際に、printf@pltの中に入っていくと、printfの関数が呼ばれるまでにいくつかの関数が呼ばれていて、レジスタには時折"GLIB_C2.0"や"printf"などの文字列が保存されていました。これによって、ライブラリの確認と、printfの参照が行われているのではないかと考えられます。

次に、printfの中身を確認します。

```
$ disas printf
Dump of assembler code for function printf:
   0xf7e54410 <+0>:	push   ebx
   0xf7e54411 <+1>:	sub    esp,0x18
   0xf7e54414 <+4>:	call   0xf7f2fb2b
   0xf7e54419 <+9>:	add    ebx,0x15ebe7
   0xf7e5441f <+15>:	lea    eax,[esp+0x24]
   0xf7e54423 <+19>:	mov    DWORD PTR [esp+0x8],eax
   0xf7e54427 <+23>:	mov    eax,DWORD PTR [esp+0x20]
   0xf7e5442b <+27>:	mov    DWORD PTR [esp+0x4],eax
   0xf7e5442f <+31>:	mov    eax,DWORD PTR [ebx-0x70]
   0xf7e54435 <+37>:	mov    eax,DWORD PTR [eax]
   0xf7e54437 <+39>:	mov    DWORD PTR [esp],eax
   0xf7e5443a <+42>:	call   0xf7e4a810 <vfprintf>
   0xf7e5443f <+47>:	add    esp,0x18
   0xf7e54442 <+50>:	pop    ebx
   0xf7e54443 <+51>:	ret    
End of assembler dump.
```
call 0xf7f2fb2bとあるので、ステップインで中に入ってみてみると、

```
 0xf7f2fb2b:	mov    ebx,DWORD PTR [esp]
 0xf7f2fb2e:	ret
```

ebxにespに退避させられたリターンアドレスを保存してretしています。ret先の命令にはadd ebx, 0x15ebe7とあるので、何らかのオフセット値を利用していることがわかります。その後に続く命令で、ebxからオフセット0x70を引いたアドレスをvfprintfの引数として渡しています。この値が何か調べるために、0xf7fb3000の周辺がメモリマップ中でどこに位置しているのか調べ、またここで渡しているのはどんな値なのか調べてみました。

```
$ shell cat /proc/7451/maps
f7e07000-f7fb1000 r-xp 00000000 08:01 933878                             /lib/i386-linux-gnu/libc-2.19.so
f7fb1000-f7fb3000 r--p 001aa000 08:01 933878                             /lib/i386-linux-gnu/libc-2.19.so
f7fb3000-f7fb4000 rw-p 001ac000 08:01 933878                             /lib/i386-linux-gnu/libc-2.19.so
$ p $ebx
$1 = 0xf7fb3000
$ p $ebx-0x70
$2 = 0xf7fb2f90
$ x/xw $2
f7fb2f90:	0xf7fb3d80
$ x/xw 0xf7fb3d80
0xf7fb3d80 <stdout>:	0xf7fb3ac0
$ x/xw 0xf7fb3ac0
0xf7fb3ac0 <_IO_2_1_stdout_>:	0xfbad2284

```

以上のことと、残り2つの引数がそれぞれ順に"Hello, %s\n"とargv[0]であることから、これをC言語で書き直すと`vfprintf(_IO_2_1_stdout_, "Hello, %s\n", argv[0])`と書けることがわかりました。それでは、次はvfprintf関数の中に入っていきます。ここで一度、現在どの関数の中まで入っているのかを確認しておきます。

```
$ where
#0  0xf7e4a81c in vfprintf () from /lib/i386-linux-gnu/libc.so.6
#1  0xf7e5443f in printf () from /lib/i386-linux-gnu/libc.so.6
#2  0x0804843b in main (argc=0x1, argv=0xffffd524) at helloargv0.c:5
#3  0xf7e20af3 in __libc_start_main () from /lib/i386-linux-gnu/libc.so.6
#4  0x08048341 in _start ()
```

vfprintfでもまた0xf7f2fb2bが呼び出されていました。その後、先程と同様にebx値が加算されていたので見てみると、先程と同じ0xf7fb3000となっていました。つまり、この0xf7f2fb2bを含む一連のステップによって、ebxにglibcのアドレスがロードされたアドレスが保存されることがわかりました。

何度かレジスタに値を代入して判定を行った後、途中でジャンプする箇所がありました。そのジャンプした先の中ではstrchrnul("Hello, %s", 0x25)が呼ばれており、ASCII文字で0x25は"%"です。この関数は第一引数に第二引数が含まれればそのポインタが、もしなければ末尾のヌル文字のポインタを返します。つまり、これはフォーマット文字列中に最初に出てくるフォーマット記述子がどこにあるのかを調べています。ここで何をしているかわかったので、先に進みたいと思います。処理を進めていくと、call DWORD PTR [eax+0x1c]という関数呼び出し命令がありました。

```
$ x/xw $eax+0xc
0xf7fb2abc:	0xf7e76270
$ x/wx 0xf7e76270
0xf7e76270 <_IO_file_xsputn>:	0x57c03155
```
`_IO_file_xsputn`の中でも先程同様にlibcのアドレスをレジスタに保存している処理がありました。その直後にまたstdoutをediに保存しているので、文字列出力に関係する関数であると予想できます。

いくつかの処理の後、また"call DWORD PTR [eax+0xc]"という関数呼び出しが行われているので、そのポインタの中身を調べてみると、`_IO_file_overflow`という関数であることがわかりました。引数とともにC言語風に書くと、`_IO_file_overflow(stdout, 0xffffffff);`という風に書けます。

```
$ x/wx $eax+0xc
0xf7fb2aac:	0xf7e76f20
$ x/wx 0xf7e76f20
0xf7e76f20 <_IO_file_overflow>:	0x53565755
```

`_IO_file_overflow`の中に入ってみて逆アセンブルしたものを眺めると、これまでの関数と同様に最初にスタックフレームの構築、libcのアドレスを求める処理、その後にいろいろな値のチェックとジャンプ系の命令が並んでいました。ステップ実行させていくと、2度のジャンプの後に`_IO_doallocbuf`という関数がstdoutを引数として呼び出されていました。名前からするに、この関数はメモリ確保用の関数っぽいので飛ばして先に進むことにしました。

何度かのジャンプの後に、`_IO_do_write`という関数が呼び出されていました。引数を入れてC言語風に書くと`_IO_do_write(_IO_2_1_stdout_, *0xf7fd5000, 0);`です。第二引数のポインタを調べてみたのですが、何のアドレスなのかわかりませんでした。メモリマップを見ても、その周辺にはデータが入っていないようです。ちなみに、"*"の記号は、ポインタという意味で使っています。

次は、`_IO_do_write`の中に入って調べてみることにしましたが、特に何も起こらずにジャンプしてretしました。

結局、`_IO_file_xsputn`まで戻ってきました。`_IO_default_xsputn`という関数が呼び出されたので、今度はこの中に入ってみました。引数は `_IO_default_xsputn(_IO_2_1_stdout_, "Hello, %s\n", 0x7 )`です。この内部でcall DWORD PTR [eax+0c]により関数呼び出しが行われていました。このアドレスを調べると、先程も呼ばれていた`_IO_file_overflow`であることがわかりました。

```
$ x/xw $eax+0xc
0xf7fb2aac:	0xf7e76f20
gdb-peda$ x/xw 0xf7e76f20
0xf7e76f20 <_IO_file_overflow>: 0x53565755
```

しかし、引数を見てみると、先程とはすこし違っていて、引数とともに書くと`_IO_file_overflow(_IO_2_1_stdout_, 0x48 (b'H'))`という風になります。関数の中にはいって見ると、特に何も起こらずretしました。レジスタを見ると、esiにもとの文字列から最初の文字"H"が取り除かれた文字列が保存されています。

```
$ x/s $esi
0x80484e1:	"ello, %s\n"
```
そして、関数から戻ってくると、先程と同じ場所で次は`_IO_file_overflow(_IO_2_1_stdout_, 'e');`が呼び出されました。見てみると、`<_IO_default_xsputn+48>`から`<_IO_default_xsputn+156>`にかけてループすることで文字列を１文字列ずつ読み込んでいるようです。同じ処理なので、ずっと処理を進めていくと、`IO_default_xputn`から戻ってきました。このとき、eaxには0x7が保存されており、そういえばこの0x7という数字は、`_IO_default_xsputn(_IO_2_1_stdout_, "Hello, %s\n", 0x7 )`で引数として使われていました。また、"Hello, "の文字数は7文字です。つまり、`_IO_default_xsputn`は第一引数のファイルストリームに第三引数の文字数分書き込む関数だったことがわかり、また先程から引数に使われている`_IO_2_1_stdout_`は最後に出力したい文字列の配列ではないかと推測できます。その後は特に何も起こらず、`_IO_file_xsputn`も抜け出し、vfprinfまで帰ってきました。その後、正常に処理が完了したか文字数のチェックが行われ、その後、残った"%s\n"に対する処理が行われます。

まず、最初の"%"を無視した文字列を取り出してローカル変数に保存し、その次のフォーマット指定子の2文字目、つまり今回で言えば"s"の部分をeaxに格納します。その文字を大文字にしてedxに格納し、それと0x5a('Z')と比較することで、まずフォーマット指定子の文字の範囲内にあるかどうかをチェックします。これは後からglibc-2.19のソースを確認したのですが、たしかにvfprintf.cの中にフォーマット指定子のテーブルがありました。

その後、arg[0]を先頭から順に読み込みます。このとき、以下の命令を使って、arg[0]の文字数を調べます。eaxは0に、ecxは0xffffffffに設定されているので、scas命令によってEFLAGSレジスタのZFフラグが更新されるまでrepnz命令は行われ、その後にnot ecxすることで文字列終端のヌル文字を含む文字列の長さが得られます。

```
<vfprintf+18771>:	repnz scas al,BYTE PTR es:[edi]

```

後は"Hello, %s\n"のときと同様で、`_IO_file_xsputn(_IO_2_1_stdout_, argv[0], strlen(argv[0]));`が呼び出されます。ここで、strlenは使用していないのですが、ヌル文字を除く文字列の長さを表すのに書きました。


残った文字列"\n"に関しても、先程の"Hello %s\n"と同様に`_IO_file_xsputn`が呼び出されます。
strchrnulでフォーマット文字、もしくはヌル文字のポインタを取得します。その後、`_IO_file_xsputn(_IO_2_1_stdout_, "\n", 1)`が呼ばれます。再度`_IO_file_xsputn`の中に入ってみると、途中で見慣れない関数が呼び出されていました。0xfe83ae0という関数です。ポインタが何を指しているのか調べてみても名前がわからなかったので、中に入ってみることにしました。ここで、一度whereコマンドでどこまで入り込んでいるのか確認します。

```
$ where
#0  0xf7e83ae0 in ?? () from /lib/i386-linux-gnu/libc.so.6
#1  0xf7e762d5 in _IO_file_xsputn () from /lib/i386-linux-gnu/libc.so.6
#2  0xf7e4ad82 in vfprintf () from /lib/i386-linux-gnu/libc.so.6
#3  0xf7e5443f in printf () from /lib/i386-linux-gnu/libc.so.6
#4  0x0804843b in main (argc=0x1, argv=0xffffd524) at helloargv0.c:5
#5  0xf7e20af3 in __libc_start_main () from /lib/i386-linux-gnu/libc.so.6
#6  0x08048341 in _start ()
```

中に入ってみると、'\n'の1文字文だけ_IO_2_1_stdoutに書き込む関数でした。また`_IO_file_xsputn`に帰ってきました。これですべての文字列を読み込んだはずなので、後は出力するだけのはずです。

また`_IO_file_overflow(_IO_2_1_stdout_, 0xffffffff)`が呼ばれているので中に入ってみます。すると、`_IO_do_write(_IO_2_1_stdout_, "Hello, (argv[0]の中身)\n", 0x3c)`が呼ばれているので、中に入ってみます。ところで、出力されるはずの文字列の長さを調べてみると、改行文字を含めて0x3cであることがわかりました。これが文字列表示をしているところのようです。`_IO_do_write`の中で、先程のcall 0cf7e83ae0が実行されています。これは先程は_IO_2_1_stdout_に引数の文字数文だけ書き込んでいました。今回は`3b`が引数になっています。この中に入って調べてみることにしました。今度は、call DWORD PTR [eax+0x3c]という命令が実行されています。先程は見なかった命令です。

```
$ x/wx $eax+0x3c
0xf7fb2adc:	0xf7e75cf0
$ x/xw 0xf7e75cf0
0xf7e75cf0 <_IO_file_write>:	0x83565755
```
引数とともに書くと、`_IO_file_write(stdout, "Hello, (argv[0])\n", 0x3c)`となります。更にこの関数の中で、`write(1, "Hello, argv[0],\n", 0x3c)`がよれています。更にこの中にも入っていきます。writeの内部ではcall DWORD PTR gs:0x10が呼ばれています。このgs:10はgsセグメントのオフセット10という意味ですが、ここには仮想システムコールという、システムコールを呼び出す関数である`__kernel_vsyscall`のアドレスがあります。システムコールはOSの機能呼び出しに使われる命令のことです。このことから、今呼ばれているのは`__kernel_vsyscall`であることがわかります。更に深掘っていくと、`__kernel_vsyscall`の内部でsysenterという命令が実行されています。引数として、32bitのLinux場合だとeaxはシステムコールの番号,ebxに第1引数,ecxに第2引数,edxに第3引数が渡されます。システムコール番号が4なので、今呼び出すシステムコールはwrite,writeシステムコールは引数が順にファイル記述子、バッファのアドレス、データサイズとなっており、標準入出力stdoutは1に対応しているので、ここで標準出力に出力されます。

```
$ ni
Hello, /home/ubuntu/Projects/AnalysingPrintf/src/helloargv0
```
