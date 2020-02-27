
### 版本
* ubuntu 16.04
* linux-kernel 4.4.1

<a href="https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/" target="_blank" rel="noopener">https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/</a>
<a href="https://elixir.bootlin.com/linux/v4.4.100/source" target="_blank" rel="noopener">https://elixir.bootlin.com/linux/v4.4.100/source</a>

### 環境

下載後先解壓縮

``` 
$ tar zxvf linux-3.9.1.tar.gz 
```

編譯 
```
$ cd linux-3.9.1
$ make menuconfig # or make defconfig
$ make clean
$ make
$ make modules_install install
```
第一次make有噴 

> error: include/linux/compiler-gcc.h:106:30: fatal error: linux/compiler-gcc5.h：No such file or directory

把當前kernel原碼中的gcc複製到要編譯的kernel裡即可
> <a href="https://blog.csdn.net/u014525494/article/details/53573298" target="_blank" rel="noopener">https://blog.csdn.net/u014525494/article/details/53573298</a>


```
$ file arch/x86/boot/bzImage
arch/x86/boot/bzImage: Linux kernel x86 boot executable bzImage, version 3.9.1 (root@eugene) #0 SMP Sun Nov 17 22:21:14 CST 2019, RO-rootFS, swap_dev 0x5, Normal VGA
```

Enable the kernel for boot(雖然好像他會自己加)

```
$ sudo update-initramfs -c -k 3.19
$ sudo update-grub
```

然後直接用這個kernel開機的話會卡在
`Uncompressing Linux... done, booting the kernel.`
上網查了之後發現可能的原因一大堆，索性直接換了3.10.1看看結果還是一樣
另外發現在 `make install` 和 `update-initramfs` 的時候有噴 `amd64-microcode unsupported kernel version` 
最後改成4.4.1(本來的kernel是4.4.0)就好了

開機之後確定使用的kernel是我們剛剛編的
```
$ uname -a
Linux eugene 4.4.1 #1 SMP Mon Nov 18 14:36:16 CST 2019 x86_64 x86_64 x86_64 GNU/Linux
```

成功!


### 新增一個system call

* 先建立一個程式裡面有 `sys_helloworld` 的 function
```
$ cd linux-4.4.1/
$ mkdir mycall
$ vim helloworld.c

#include <linux/kernel.h>
asmlinkage int sys_helloworld(void){
        printk("hello world");
        return 0;
}
```

* 建立Makefile
```
$ vim Makefile

obj-y := helloworld.o
```

* 在主要的Makefile中新增mycall資料夾進去
```
$ cd ..
$ vim Makefile

core-y += kernel/ certs/ mm/ fs/ ipc/ security/ crypto/ block/ mycall/
```

* 在system call table裡新增我們的system call

```
$ vim arch/x86/entry/syscalls/syscall_64.tbl

546 64 helloworld sys_helloworld # 在最後面一行加上
```

* 修改 system call header

```
$ vim include/linux/syscalls.h

# 加在#endif前
asmlinkage int helloworld(void);
```

* 編譯

```
$ sudo make
$ sudo make modules_install install
$ reboot
```

最後寫一個程式測試一下
```
#include <stdio.h>
#include <syscall.h>
#include <sys/types.h>

int main(){
        int a = syscall(546);
        printf("system call sys_hello_world return %d\n", a);
        return 0;
}

```

```
$ dmesg
```
有看到hello world就代表成功了

<p>reference:<br>
<a href="https://chybeta.github.io/2017/10/19/Linux-kernel-development-1-%E7%8E%AF%E5%A2%83%E5%87%86%E5%A4%87/" target="_blank" rel="noopener">https://chybeta.github.io/2017/10/19/Linux-kernel-development-1-环境准备/</a><br>
<a href="https://www.linux.com/tutorials/how-compile-linux-kernel-0/" target="_blank" rel="noopener">https://www.linux.com/tutorials/how-compile-linux-kernel-0/</a><br>
<a href="https://wenyuangg.github.io/posts/linux/linux-add-system-call.html" target="_blank" rel="noopener">https://wenyuangg.github.io/posts/linux/linux-add-system-call.html</a><br>
<a href="https://blog.kaibro.tw/2016/11/07/Linux-Kernel%E7%B7%A8%E8%AD%AF-Ubuntu/?fbclid=IwAR0n1xjghssrijlA7L8nFhjojsu-Wdb8w25900l_WVtvDQeJgJzv7MaXxIU" target="_blank" rel="noopener">https://blog.kaibro.tw/2016/11/07/Linux-Kernel編譯-Ubuntu/?fbclid=IwAR0n1xjghssrijlA7L8nFhjojsu-Wdb8w25900l_WVtvDQeJgJzv7MaXxIU</a></p>