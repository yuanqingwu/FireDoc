https://www.linuxcool.com/


### rm
Linux rm命令用于删除一个文件或者目录。

###### 语法
```
rm [options] name...
```
###### 参数
- -i 删除前逐一询问确认。
-  -f 即使原档案属性设为唯读，亦直接删除，无需逐一确认。
-  -r/R  将目录及以下之档案亦逐一删除。递归删除

##### 实例
删除文件可以直接使用rm命令，若删除目录则必须配合选项"-r"，例如：
```
touch test.txt
rm  test.txt 

mkdir test
rm  test  
rm: 无法删除目录"test": 是一个目录  
rm  -r  test  
```
### rmdir
Linux rmdir命令删除空的目录。

###### 语法
```
rmdir [-p] dirName
```

###### 参数
- -p 是当子目录被删除后使它也成为空目录的话，则顺便一并删除。

###### 实例
```
rmdir test/

在工作目录下的 BBB 目录中，删除名为 Test 的子目录。若 Test 删除后，BBB 目录成为空目录，则 BBB 亦予删除。

rmdir -p BBB/Test
```


