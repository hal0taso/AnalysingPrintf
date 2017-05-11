# README

実行環境は以下の通りである。

```
$ uname -a
Linux hal0taso 4.10.0-20-generic #22-Ubuntu SMP Thu Apr 20 09:22:42 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
$ gcc --version
gcc (Ubuntu 6.3.0-12ubuntu2) 6.3.0 20170406

```

まず、以下のようなプログラムを作成した。

```
#include <stdio.h>

int main(int argc, char *argv[]){
  
  printf("Hello, World!\n");

  return 0;
}
```

実行すると、


```
$ ./helloworld 
Hello, World!
```

