---
layout: post
title:  "YAVC"
subtitle: "Yet anoter vim config"
date:   2021-04-20 23:30:00 +0000
share-img: "/img/vim/Vimlogo.png"
image: "/img/vim/Vimlogo.png"
tags: [tools, python]
---
I've recently spent a bit of time looking at ways to improve my python development
experience. I've been a long time vim user and wondered if an IDE like the popular
[pycharm](https://www.jetbrains.com/pycharm/) from JetBrains was worth a try. I have
some co-workers that use it and I figured it was time to try it out. As you can tell
by the title of this post, I really wasn't impressed. It took a long time to load,
had refresh issues (could be that I use i3-regolity), and I knew the learning curve
for the editor was going to be pretty extreme coming from years of vim. 

However, the ability to go to definitions was a pretty nice feature so I decided to
see if I could get that without changing my primary editor. It turns out you can get
that and much more. I'll go over what I'm using and how to get started in this post.

## Language Servers
I was aware of the concept of a Language Server but had not taken the time to see
what it involved. The short explanation is that a language server is a program that
knows how to provide advanced features for a specific programing language. The
features include things like refactoring, code completion, highlighting, and go to
definition. It knows a language and it knows how to navigate and modify that
language. A language server is the brains behind the advanced features that people
like about an IDE. In fact, it was developed by Microsoft for Visual Studio Code.

Language servers communicate to clients by the Language Server Protocol (lsp). I've
found that [langserv.org](langserv.org) provides a nice overview of available
servers and clients. 

## Vim Language Server Clients
Since I'm using [NeoVim](https://neovim.io/) I was looking for vim compatible
clients, and there are a few options. Technically, I don't use vim, I use NeoVim but
if you're a vim user you might want to try it out it was a fork of Vim 7 and at
least for my use it was a drop in replacement. Some plugins may not be compatible
between them, but when I switched NeoVim's asynchronous plugins made a noticeable
difference in vim performance. This article will be for NeoVim YMMV for Vim.

## Vim Plugins
### Plug
To install plugins I'm using [vim-plug](https://github.com/junegunn/vim-plug). It's
github page has the install directions, at the time of this writting for NeoVim you
can install it with.

```bash
sh -c 'curl -fLo "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/autoload/plug.vim --create-dirs \
       https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim'
```
After that all of the following plugin configurations have to go in the following
block of init code.

```vim
call plug#begin()
  Addons Here
call plug#end()
```
That's it, minimal plugin manager with a simple syntax and support for parallel
processing in NeoVim.

After adding a Pluing via Plug to your vim config, exit and relaunch vim, and run
`:PlugInstall` to install the new plug in. The other commands of note are:

* `PlugClean` to remove unused plug ins
* `PlugUpgrade` to upgrade plug ins

### NeoVim Config
The Vim config with plugins that I'm currently using is:

```vim
set nocompatible
filetype off

call plug#begin()
"Color scheme
Plug 'NLKNguyen/papercolor-theme'

" Show git change status in gutter
Plug 'airblade/vim-gitgutter'

" Lightline status bar
Plug 'itchyny/lightline.vim'
Plug 'maximbaz/lightline-ale'

" Vimwiki
Plug 'vimwiki/vimwiki', { 'branch': 'dev'}
Plug 'michal-h21/vimwiki-sync'

" Python
Plug 'Vimjas/vim-python-pep8-indent'

" Code browser
Plug 'preservim/nerdtree'
Plug 'Xuyuanp/nerdtree-git-plugin'
Plug 'PhilRunninger/nerdtree-buffer-ops'

" Code commenting
Plug 'preservim/nerdcommenter'

" Automatic closing
Plug 'jiangmiao/auto-pairs'

" Snippets
Plug 'SirVer/ultisnips'
Plug 'honza/vim-snippets'

" Code syntax/cleanliness checking (ALE)
Plug 'dense-analysis/ale'

" Code Autocompletion
Plug 'Shougo/deoplete.nvim', { 'do': ':UpdateRemotePlugins' }

" Tagbar
Plug 'preservim/tagbar'

call plug#end()
" Done with plugins

" Per filetype options
filetype plugin indent on
filetype plugin on

au FileType python setl shiftwidth=4 colorcolumn=120 expandtab
au FileType yml    setl shiftwidth=2 expandtab

au BufRead *.md setlocal spell
au BufRead *.txt setlocal spell
au BufRead *.markdown setlocal spell

" GUI Options
:set guioptions-=m  "remove menu bar
:set guioptions-=T  "remove toolbar
:set guioptions-=r  "remove right-hand scroll bar
:set guioptions-=L  "remove left-hand scroll bar

" Colors
" Papercolor
set background=dark
colorscheme PaperColor
let g:PaperColor_Theme_Options = { 
  \   'language': { 
  \     'python': { 
  \       'highlight_builtins' : 1 
  \      } 
  \   }
  \ }
" Lightline
let g:lightline = { 'colorscheme': 'PaperColor' }
let g:lightline.component_expand = {
      \  'linter_checking': 'lightline#ale#checking',
      \  'linter_infos': 'lightline#ale#infos',
      \  'linter_warnings': 'lightline#ale#warnings',
      \  'linter_errors': 'lightline#ale#errors',
      \  'linter_ok': 'lightline#ale#ok',
      \ }
let g:lightline.component_type = {
      \     'linter_checking': 'right',
      \     'linter_infos': 'right',
      \     'linter_warnings': 'warning',
      \     'linter_errors': 'error',
      \     'linter_ok': 'right',
      \ }
let g:lightline.active = { 'right': [[ 'linter_checking', 'linter_errors', 'linter_warnings', 'linter_infos', 'linter_ok' ]] }

" Text editing / indenting
set autoindent
set hlsearch
set ignorecase
set nowrap
set shiftwidth=4
set showmode
set wrapmargin=2

" Nerdtree
" Mirror the NERDTree before showing it. This makes it the same on all tabs.
nnoremap <C-n> :NERDTreeMirror<CR>:NERDTreeFocus<CR>

" Deoplete
let g:deoplete#enable_at_startup = 1
call deoplete#custom#option('sources', {'_': ['ale']})

" ALE
let g:ale_open_list = 1
let g:ale_list_window_size = 5
let g:ale_linters = {'python': ['flake8', 'pyls']}
let g:ale_fixers = {
\    'python': [
\	'isort',
\	'black',
\    ],
\}

let g:ale_python_auto_pipenv = 1
let g:ale_python_black_executable = "/home/$USER/.local/share/nvim/black/bin/black"
let g:black_virtualenv = "/home/$USER/.local/share/nvim/black"

" Hotkeys
map <F4> :NERDTreeToggle<CR>
map <F5> :ALEFix<CR>
map <F6> <Plug>(ale_toggle_buffer)
nmap <F8> :TagbarToggle<CR>
nmap <silent> <C-k> <Plug>(ale_previous_wrap)
nmap <silent> <C-j> <Plug>(ale_next_wrap)

" Use system python when in venv
let g:python3_host_prog="/usr/bin/python3"
let g:python_host_prog=0

" Faster gutter updates for git
set updatetime=100

" Setup hybrid line numbers
set number
set relativenumber

" Allow saving of files as sudo when I forgot to start vim using sudo.
cmap w!! w !sudo tee > /dev/null %
```

## ALE
I've been using [ALE](https://github.com/dense-analysis/ale) for asynchronous
linting of my code. I'm extremely happy with ALE's performance for linting. However,
it turns out it's *also* a very competent LSP client. In fact, ALE will
automatically load a number of LSP severs if they are installed when a file of the
appropriate type is opened. While I do use a number of plugins, if you are new to
Vim I recommend starting with ALE.

ALE is the plug in that provides most of the IDE capabilities. In the above
configuration ALE is set to use `isort` for automatic ordering of imports and
`black` for automatic code formatting. Both programs will pick up project specific
settings in `.tox` or `pyproject` files. This combination makes my code consistent
automatically.

The setting `ale_python_auto_pipenv` detects when you're using a virtual environment
and ALE will find packages appropriately. This is *extremely* useful if you're
working on a project that has dependencies that you don't have installed locally.
You can install the dependencies in an environment, activate the environment, and
then launch your vim and ALE will pickup the packages.

The settings for `black_virtualenv` and `black_executable` should be fairly straight
forward. I've installed a virtual environment which black runs in to keep it
separate from my host python version. By telling ALE where the environment is and
the executable it will use that version and you can manage it independently. I set
this up prior to my introduction to `pipx` I haven't tried it myself, but I suspect
you could skip this config if you install black with `pipx` and still manage it
independently.

My configuration uses
[python-language-server](https://github.com/palantir/python-language-server) to
provide the language server to ALE. You can install it with `pipx install
python-language-server`. ALE will automatically launch `pyls` if available when
editing a .py file.

Finally, I've mapped a few hot keys, `F5` will tell ALE to fix your code and `F6`
will close the error window if you don't want to see it currently. While linting
errors are present `<ctl>j` will move to the next error and `<ctl>k` will move to
the previous error.

## Deopelete
I was using Deoplete before I was using ALE for completion with a LSP. Deopelete has
an integration with ALE and so I've set it up to receive information form ALE and
python-language-server. I haven't done a comparison of using ALE alone, if you have
experience with both I'd be interested to hear your thoughts.

While typing, deopelete will provide autocompletion options. Pressing `<ctl>n`
iteraties forward through the completion optoins and `<ctl>p` goes back to the
prevoius completion. Completions will also open a window at the top that shows the
docstring for the function. This window can be closed and wrapping can be enabled
via `set wrap` like any other window.

## Gitgutter
A staple for anyone doing development, this plug in will show icons on the left hand
'gutter' for the current git status of each line of code. You can easily tell which
lines were added, removed, or modified.

## Lightline
I'm using a status bar, which is a bar at the bottom of VIM that shows the mode you
are in and the file you are currently editing. You can customize the status bar, but
the only extra configuration I have included is to enable a status based on the
linting. The Lightline ALE configuration sets this up and is directly out of the
Ligthline example configurations.

## VimWiki
I'm not covering VimWiki in this post, but if you take notes in Vim you might want
to check out it's [homepage](https://github.com/vimwiki/vimwiki). I use VimWiki for
work and personal notes, it's a convenient way to organize notes and sync them with
a git repository for safe keeping.

## Pep8-indent
This module also requires no configuration and helps make smart indentions as you
are coding. While ALE will automatically fix indentions I prefer to have indentions
correct in the fist place most of the time.

## Nerdtree
The [Nerdtree](https://github.com/preservim/nerdtree) plugins provide a convenient
way to browse files. I've mapped `F4` to pull up the NerdTree browser in on the left
of the window. Files can be opened from this window and their git status is
displayed as well. Pressing `F4` again closes the browser.

## Nerd commenter
[Nerd Commenter](https://github.com/preservim/nerdcommenter) provides intelligent
commenting and block commenting. For python I find that using the built in hotkey
`\cl` will comment out a selected block of text and not violate Pep8.

## Auto pairs
[auto pairs](https://github.com/jiangmiao/auto-pairs) closes brackets and comments of
all types automatically. I use to typically type both an open and close of brackets
and comments, and then backspaces to fill in the pair. This plug in makes that
happen when the first bracket or quotation is typed. It also takes care of things
like adding symmetrical spaces as you type. It's hard to overstate the value of this
plug in. It works as you expect saving you keystrokes while rarely getting in the
way.

## Ultisnips
[UltiSnips](https://github.com/SirVer/ultisnips) is a strange plug in to get use to.
I've only recently started using it, but where I do use it it's very valuable.
Ultisnips templates common code blocks and make them easy to insert. There is a
repository of templates for a number of languages. While I haven't found the python
templates very useful, they don't get in the way and the Markdown templates have at
least one shortcut I find extremely useful. Most projects have a markdown file and
Ultisnips will automatically add markdown link if you type `link<tab>`.

You can also define custom snips with Ultisnips. While it's a new plugin to me, the
community repository and ability to create custom templates make it worthing having.

## Tagbar
[Tagbar](https://github.com/preservim/tagbar) provides a 'tag' browser on the right
side of your editor. Tags provide an overview of the Classes, Methods, and
Attributes in the current file. You can jump to the tags for quick navigation. I
have Tagbar mapped to `F8` which opens and closes the Tag window.


