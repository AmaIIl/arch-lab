# arch-lab

一个很掉头发的实验，对应了csapp中第四、五章的内容。
首先是实验的前提工作，需要进行一些配置才可以进行实验：
```
sudo apt install tcl tcl-dev tk tk-dev
sed -i "s/tcl8.5/tcl8.6/g" Makefile
sed -i "s/CFLAGS=/CFLAGS=-DUSE_INTERP_RESULT /g" Makefile
cd sim/
make clean; make
```
