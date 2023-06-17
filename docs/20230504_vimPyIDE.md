
The ycmd server SHUT DOWN (restart with ':YcmRestartServer'). YCM core library not detected; you need to compile YCM before using it. Follow the instructions in the documentation.

```sh
sudo dnf install make automake gcc gcc-c++ cmake

sudo pacman -Syu make automake gcc cmake
```

--- 

```sh
cd YouCompleteMe/
ls
autoload  codecov.yml  CODE_OF_CONDUCT.md  CONTRIBUTING.md  COPYING.txt  doc  install.py  install.sh  plugin  print_todos.sh  python  README.md  run_tests.py  test  third_party  tox.ini  update-vim-docs  vimrc_ycm_minimal
git submodule update --init --recursive
./install.sh 
```

```sh
  File "/home/jeff/.vim/bundle/YouCompleteMe/third_party/ycmd/ycmd/utils.py", line 505, in ImportAndCheckCore
    ycm_core = ImportCore()
               ^^^^^^^^^^^^
  File "/home/jeff/.vim/bundle/YouCompleteMe/third_party/ycmd/ycmd/utils.py", line 493, in ImportCore
    import ycm_core as ycm_core
ImportError: /home/jeff/miniconda3/envs/gpt/lib/libstdc++.so.6: version `GLIBCXX_3.4.30' not found (required by /home/jeff/.vim/bundle/YouCompleteMe/third_party/ycmd/ycm_core.cpython-311-x86_64-linux-gnu.so)
```

https://stackoverflow.com/questions/47667119/ycm-error-the-ycmd-server-shut-down-restart-wit-the-instructions-in-the-docu

```sh
(base) [jeff@fedora YouCompleteMe]$ strings ~/miniconda3/lib/libstdc++.so.6 | grep GLIBCXX_3.4.30
(base) [jeff@fedora YouCompleteMe]$ strings ~/miniconda3/lib/libstdc++.so.6 | grep GLIBCXX_3.4.3
GLIBCXX_3.4.3
GLIBCXX_3.4.3
(base) [jeff@fedora YouCompleteMe]$ strings /usr/lib64/libstdc++.so.6.0.30 | grep GLIBCXX_3.4.30
GLIBCXX_3.4.30
(base) [jeff@fedora YouCompleteMe]$ cp /usr/lib64/libstdc++.so.6.0.30 ~/miniconda3/lib/libstdc++.so.6
(base) [jeff@fedora YouCompleteMe]$ 
```