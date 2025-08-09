# NeoVim
--- 
Recently, the stock Vim experience has left much to be desired.
I've been looking into trying a new commandline based IDE and have decided to have a crack with NeoVim

A lot of people have asked me why I perfer cmd text editors in place of a full-fledged IDE, and a lot of the time it's the snapiness of the program.
I've found that a lot of programs (VSCode) carry a lot of bloat, while TMux + VIM provides a quick command line experience for back-end operations.

# Installation
---
On the Ubuntu WSL package that I use, the NeoVim installation is as straightforward as it gets:
```
$sudo apt-get install neovim
```

# Tutor
I've decided to take a bit of a refresher on the Vim keybinds by using the `:Tutor` function
This is a built-in "Help" manual which teaches the basics of vim functionality, such as the `hjkl` navigation keys, which take a while to adjust to.


# Customization
I added the following customization items to my NeoVim conf file:
```
set nocompatible
set showmatch
set ignorecase
set mouse=v
set hlsearch
set incsearch
set tabstop = 4
set softtabstop=4
set expand tab
set shiftwidth=4
set autoindent
set number
set wildmode=longest,list
set cc=80
filetype plugin indent on
syntax on
set clipboard=unnamedplus
filetype plugin on
set cursor line
set ttyfast
```
