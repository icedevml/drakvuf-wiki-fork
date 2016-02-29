If you encounter problems with DRAKVUF, the best option is to enable the printing of additional debug information so that the root cause of your problem can potentially be spotted. For this, you will need to recompile DRAKVUF and run it again the VM you have experienced the error on.

```c
git clean -xdf
git reset --hard
./autogen.sh
./configure --enable-debug
make
```

If you suspect that a certain plugin is causing the problem, you should turn off all other plugins to better isolate the problem, which you can do with the `--disable-plugin-<pluginname>` configure option. For example:

```c
./configure --enable-debug --disable-plugin-syscalls
```

Make sure to include this debug log in any issue you open up so that others can better help you figuring out the problem.