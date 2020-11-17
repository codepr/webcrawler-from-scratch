# Development environment

Go is a compact language and it's all-around well supported on the majority of
the most popular editors and IDEs.

Personally I prefer to stick with lightweight and open-source editors, plus
with the advent of LSP servers, every editor can easily provide almost
IDE-level development experience, in case of go, beside `GoLand` and `IntelliJ`
products, the list ultimately leads to two main choices:

- `vim` (or `neovim`)
- `vscode`

In my experience both provide pretty similiar developing experience and
ultimately it comes down to personal preference, I like `neovim` +
`fatih/vim-go` + `neoclide/coc.nvim` + `gopls` language server, it works well
and provides all the necessary, from autocomplete to refactoring and
linting/code formatting.

I suggest running the handy `govalidate` tool by
[https://github.com/rakyll/govalidate](https://github.com/rakyll/govalidate) to
be sure that you have installed all you need or get suggestions on how to solve
problems if needed.
