# download and install Include What You Use

```
~/github » sudo apt-get install clang
~/github » sudo apt-get install libclang-dev
~/github » sudo apt-get install llvm-dev
~/github » git clone https://github.com/include-what-you-use/include-what-you-use.git
~/github/include-what-you-use ‹clang_10› » git checkout clang_10
~/github/include-what-you-use ‹clang_10› » cd ..
~/github » mkdir build && cd build
~/github/build » sudo cmake -G "Unix Makefiles" -DCMAKE_PREFIX_PATH=/usr/lib/llvm-10 -DCMAKE_INSTALL_PREFIX=/usr ../include-what-you-use
~/github/build » sudo make
~/github/build » sudo make install
```
# use iwyu to clean isulad

```
~/openeuler_iSulad/build ‹image*› » sudo CC="clang" CXX="clang++" cmake -DCMAKE_CXX_INCLUDE_WHAT_YOU_USE="/usr/bin/include-what-you-use;-Xiwyu;any;-Xiwyu;iwyu;-Xiwyu;args"  -DCMAKE_C_INCLUDE_WHAT_YOU_USE="/usr/bin/include-what-you-use;-Xiwyu;any;-Xiwyu;iwyu;-Xiwyu;args" ..

~/openeuler_iSulad/build ‹image*› » sudo make -k CXX=/usr/bin/include-what-you-use 2> /tmp/iwyu.out

```
# apply fixes by iwyu python script

```
~/openeuler_iSulad/build ‹image*› » cp /home/lifeng/github/include-what-you-use/fix_includes.py ./

~/openeuler_iSulad/build ‹image*› » python fix_includes.py < /tmp/iwyu.out
```