---
Date: 2026-05-09
tags:
---
# Overview
kernel을 공부하기 위해서 필수적으로 해줘야 하는 vim setting에 대해서 정리한다.
# Contents
## why vim?
지금까지 여러 도구를 활용해 개발을 해왔을 것인데, 그 핵심은 LSP(Language Support Protocol)였다. 이 LSP는 현재 보고 있는 텍스트에 대하여 구문 분석을 지속적으로 수행하며 문법적 오류를 검출하고, 심볼의 정의 및 그 참조 관계를 저장하여 개발자에게 편한 코드 탐색을 도왔다. 그러나, 이런 LSP는 커널 레포에서는 동작하지 않는다.

리눅스 커널은 그 소스 크기가 너무나도 큰 나머지 일반적인 LSP로는 처리가 불가능하다. 보통 대규모 repo의 경우, clangd와 같은 LSP를 사용하는데, 이 clangd 조차도 버거운 것이 linux kernel이다.

그렇다면, 과거 커널 개발자들은 어떻게 지금까지 개발을 해왔던 것일까? 그들은 전통적으로 catg와 cscope를 사용했다. 이 둘은 vim에 최적화된 심볼 추적기로, 지속적으로 구문 분석을 수행해야 하는 LSP와 다르게, 단순한 symbol의 추적을 제공한다. 

커널은 개발 도구조차 장벽이 존재한다. 이 얼마나 무서운 일인가?
## ctags
ctags는 심볼 추적을 제공한다. 보다 정확하게는 심볼의 정의에 대한 참조(reference)를 제공한다. 기존 IDE에 비유하자면 "정의로 이동"에 가깝다.

동작 방식은 source를 미리 해석하여 심볼의 위치를 저장한 인덱스 파일을 준비, 이를 탐색 요청이 발생하면 탐색하는 방식이다. LSP는 백그라운드에서 코드 변화에 따른 인덱스 갱신을 지속적으로 하기에 느린 것이다.
> 커널 root에서 'make tags' 로 인덱스 파일을 생성할 수 있다.
### 핵심 키

```text
Ctrl-]       정의로 이동 (go to definition)
Ctrl-t       이전 위치로 복귀 (pop stack)

Ctrl-o       vim의 기존 기능, 커서를 이전 위치로 이동

:tag symbol     특정 symbol로 점프
:ts symbol      후보 목록 보기 (tag select)
:tn             다음 후보
:tp             이전 후보
```
## cscope
cscope는 ctag의 단점을 보완한다. ctag는 심볼을 찾으면, 이 심볼의 정의 위치로 이동한다. 그러나 어떤 심볼의 정의에서 이를 **역참조할 수 없다.**

이러한 심볼의 역참조를 포함하여 symbol 사이의 관계 일체를 분석해서 제공하는 것이 cscope이다. 이 또한 일종의 인덱스 파일이 필요하다.

> 기본 vim에는 cscope를 위한 설정이 존재하지 않는 경우가 있다. vim을 교체해줘야 한다.

> 커널 root에서 'make cscope' 로 인덱스 파일을 생성할 수 있다.
### 핵심 키
```vim
:cs find g symbol // global, 정의 찾기
:cs find c symbol // caller, 역참조, 나를 호출하는 대상
:cs find d symbol // callee, 참조, 내가 호출하는 대상
:cs find s symbol // symbol, 심볼 텍스트 전체 탐색
```

## .vimrc
앞서 설명한 기능을 vim에 설정하면 다음과 같다. 필요에 따라서 다음을 이용한다.
``` text
// 기존 vim 설정에 아래를 추가한다.
" =========================
" ctags
" =========================
set tags=./tags;,tags;

" =========================
" cscope
" =========================
if has("cscope")
  set cscopequickfix=s-,c-,d-,i-,t-,e-

  function! LoadCscope()
    let db = findfile("cscope.out", ".;")
    if !empty(db)
      let path = fnamemodify(db, ":p:h")
      execute "silent! cs add " . fnameescape(db) . " " . fnameescape(path)
    endif
  endfunction


  augroup my_cscope
    autocmd!
    autocmd VimEnter * call LoadCscope()
  augroup END

  nnoremap <leader>cs :cs find s <C-R>=expand("<cword>")<CR><CR>
  nnoremap <leader>cg :cs find g <C-R>=expand("<cword>")<CR><CR>
  nnoremap <leader>cc :cs find c <C-R>=expand("<cword>")<CR><CR>
  nnoremap <leader>ct :cs find t <C-R>=expand("<cword>")<CR><CR>
  nnoremap <leader>ce :cs find e <C-R>=expand("<cword>")<CR><CR>
  nnoremap <leader>cf :cs find f <C-R>=expand("<cfile>")<CR><CR>
  nnoremap <leader>ci :cs find i <C-R>=expand("<cfile>")<CR><CR>
  nnoremap <leader>cd :cs find d <C-R>=expand("<cword>")<CR><CR>
endif

" =========================
" ripgrep
" =========================
if executable("rg")
  set grepprg=rg\ --vimgrep\ --smart-case\ --hidden
  set grepformat=%f:%l:%c:%m
endif

" =========================
" Keymaps
" =========================
nnoremap <leader>q :copen<CR>
nnoremap <leader>n :cnext<CR>
nnoremap <leader>p :cprev<CR>
nnoremap <leader>qc :cclose<CR>

nnoremap <leader>g :grep <C-R>=expand("<cword>")<CR><CR>:copen<CR>

" fzf, 파일 시스템 탐색
nnoremap <leader>f :Files<CR>
nnoremap <leader>b :Buffers<CR>

" 심볼 탐색, 존재하는 파일 찾기
nnoremap <leader>r :Rg<CR>

" 사용 명령어 기록
nnoremap <leader>h :History<CR>

" fzf와 유사, 파일 시스템 탐색
nnoremap <leader>e :NERDTreeToggle<CR>

" =========================
" for 'gf' to follow header
" =========================
set path+=include
set path+=include/generated
```


---
# Summary
## Key mappings
###  ctags (정의 점프용)

```text
✔ 함수/구조체 정의 보러 갈 때
✔ 빠르게 jump & return 반복할 때

함수 위에서 Ctrl-]
→ 코드 읽기
→ Ctrl-t/o로 복귀
```
### cscope (관계 분석용)
```text
<leader>cg    정의 찾기 (global definition)
<leader>cc    caller 찾기 (누가 이 함수 호출함?)
<leader>cd    callee 찾기 (이 함수 안에서 뭐 호출함?)
<leader>cs    심볼 전체 검색
<leader>ct    텍스트 검색
<leader>ci    include 관계 (누가 include함?)
```
결과는 quickfix로 나옴
### ripgrep (rg) — 심볼 전체 검색
```text
<leader>g    :grep keyword
```
결과는 quickfix로 나옴
### fzf (파일 / 검색)
검색형 파일시스템 탐색기

```text
<leader>f    파일 찾기
<leader>b    버퍼 찾기
<leader>r    ripgrep 검색 (interactive, 즉각 재검색)
<leader>h    히스토리
```
### quickfix 활용 (핵심)
기능들 중에는 그 결과를 quickfix에 제공하는 경우도 있음
- cscope
- grep / ripgrep
```text
<leader>q     :copen
<leader>n     :cnext
<leader>p     :cprev
<leader>qc    :cclose
```
### 파일 이동
```text
gf        커서 아래 파일로 이동
```