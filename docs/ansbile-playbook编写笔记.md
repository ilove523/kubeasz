## ansible register 用户注意事项
> 注意：
>  1. register变量的命名不能用中横线(`-`)，比如dev-sda6_result，否则会被解析成`sda6_result`，dev会被丢掉，所以不要用`-`
> 2. ignore_errors这个关键字很重要，一定要配合设置成True，否则如果命令执行不成功，即 echo $?不为0，则在其语句后面的ansible语句不会被执行，导致程序中止。
> > 参考：https://www.cnblogs.com/gaoyuechen/p/7776373.html
