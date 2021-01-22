#1. VIM常用配置
```sh
set nocompatible
set showmode
set showcmd
set encoding=utf-8
set t_Co=256
filetype indent on

set autoindent
set tabstop=4
set expandtab
set softtabstop=4
set number
set cursorline
set textwidth=80
set wrap
set linebreak
set wrapmargin=2
set scrolloff=15
set sidescrolloff=5
set laststatus=2
set ruler
set smartcase
set visualbell
```

#2. VIM常用命令

##1. 多行注释：
1. 首先按esc进入命令行模式下，按下Ctrl + v，进入列（也叫区块）模式;
2. 在行首使用上下键选择需要注释的多行;
3. 按下键盘（大写）“I”键，进入插入模式；
4. 然后输入注释符（“//”、“#”等）;
5. 最后按下“Esc”键。
5. 注：在按下esc键后，会稍等一会才会出现注释，不要着急~~时间很短的
              
##2. 删除多行注释：
1. 首先按esc进入命令行模式下，按下Ctrl + v, 进入列模式;
2. 选定要取消注释的多行;
3. 按下“x”或者“d”.
4. 注意：如果是“//”注释，那需要执行两次该操作，如果是“#”注释，一次即可


##3. 多行删除

1. 首先在命令模式下，输入“：set nu”显示行号；
2. 通过行号确定你要删除的行；
3. 命令输入“：32,65d”,回车键，32-65行就被删除了，很快捷吧
4. 如果无意中删除错了，可以使用‘u’键恢复（命令模式下）
```
