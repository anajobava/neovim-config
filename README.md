# Neovim Configuration

This configuration is a **literate document**: the code lives here in `README.md`
as fenced code blocks, and a script called `tangle.lua` extracts those blocks in
order to produce `init.lua`, the file Neovim actually loads.

The idea is borrowed from the Emacs community (org-babel). Every plugin and
setting has its explanation right next to the code, so the config stays readable
even months later. GitHub renders this file for free. Never edit `init.lua`
directly; this file is the single source of truth.

## How It Works

Every ` ```lua ` fenced block in this file gets extracted into `init.lua` in
the order it appears. Blocks marked ` ```lua notangle ` are skipped: they
display with syntax highlighting but are not included in the output. Use them
for example code or temporarily disabled snippets.

Blocks marked ` ```sh install ` are extracted into `dependencies.sh`, a shell
script you run once on a fresh machine to install all system dependencies.
Each install block lives next to the section that needs it, so the reason for
every dependency is always documented in context.

## Prerequisites

Install Neovim >= 0.11 manually. Ubuntu's apt version is too old; use the
[official release](https://github.com/neovim/neovim/releases/latest).
Everything else is handled by `dependencies.sh` after the first tangle.

```sh install
# Core tools: git (lazy.nvim bootstrap), curl, gcc + make (parser/plugin builds)
PKG git curl gcc make
```

## Bootstrap

Install Neovim, clone this repo, then run three commands:

```sh
git clone <your-repo-url> ~/.config/nvim
nvim -l ~/.config/nvim/tangle.lua       # generates init.lua + dependencies.sh
bash ~/.config/nvim/dependencies.sh     # installs ripgrep, fd, clangd, pylsp...
nvim                                    # plugins auto-install on first launch
```

Saving this file inside Neovim re-tangles automatically. If you add a new
dependency later, add an `sh install` block next to the relevant section and
run `bash dependencies.sh` again. Already-installed packages are skipped.

After that, saving this file inside Neovim auto-tangles on every write.

---

## Auto-Tangle on Save

When you save `README.md` inside Neovim this autocmd fires, runs `tangle.lua`,
syntax-checks the output, and replaces `init.lua` only if the check passes.
On a syntax error you get a notification and the live `init.lua` is left
untouched.

The `fnamemodify` comparison normalises the path so the autocmd fires
regardless of how the buffer was opened (relative path, symlink, etc.).

```lua
vim.api.nvim_create_autocmd("BufWritePost", {
    group   = vim.api.nvim_create_augroup("Tangle", { clear = true }),
    pattern = "*",
    callback = function(args)
        local cfg    = vim.fn.stdpath("config")
        local readme = vim.fn.resolve(vim.fn.fnamemodify(cfg .. "/README.md", ":p"))
        if vim.fn.resolve(vim.fn.fnamemodify(args.file, ":p")) == readme then
            dofile(cfg .. "/tangle.lua")
        end
    end,
})
```

---

## Leader Key

The leader key must be set before lazy.nvim loads. Lazy reads it during setup
to register plugin keybindings; if it were set after, those bindings would
use the wrong leader.

`Space` is the standard modern choice: it is the largest key on the keyboard,
unused by any default Vim command, and comfortable to hit with either thumb.
`maplocalleader` is used by some filetype plugins for buffer-local bindings.

```lua
vim.g.mapleader      = " "
vim.g.maplocalleader = "\\"
```

---

## Editor Options

Standard editing behaviour. Notes on the less-obvious ones:

- `updatetime = 100`: how quickly gitsigns and LSP diagnostics respond to
  cursor movement. 100ms is snappy without being expensive on the CPU.
- `timeoutlen = 300`: how long Neovim waits after a partial keymap sequence
  before giving up. Affects how fast `jk` (our Escape) triggers and how
  quickly the which-key popup appears.
- `signcolumn = "yes"`: keeps the gutter permanently visible so the buffer
  does not jump left and right as git signs and diagnostics appear.
- `formatoptions = ""`: stops Neovim from auto-wrapping lines or
  auto-inserting comment leaders when you press Enter inside a comment.
- `relativenumber`: shows line distances instead of absolute numbers, making
  `5j`, `3k` jumps trivial to calculate at a glance.

```lua
vim.opt.number         = true
vim.opt.relativenumber = false
vim.opt.cursorline     = true
vim.opt.termguicolors  = true
vim.opt.signcolumn     = "yes"
vim.opt.tabstop        = 4
vim.opt.shiftwidth     = 4
vim.opt.softtabstop    = 4
vim.opt.expandtab      = true
vim.opt.shiftround     = true
vim.opt.autoindent     = true
vim.opt.wrap           = false
vim.opt.autowrite      = false
vim.opt.formatoptions  = ""
vim.opt.mouse          = "a"
vim.opt.updatetime     = 100
vim.opt.timeoutlen     = 300
```

---

## Clipboard

Sync the system clipboard both ways. `unnamed` covers the `*` register
(primary selection on Linux, i.e. highlighted text). `unnamedplus` covers the
`+` register (the clipboard you fill with Ctrl+C). Together they keep
copy-paste between Neovim and other apps in sync.

```lua
vim.opt.clipboard = "unnamed,unnamedplus"
```

---

## Spell Check

Enable spell checking automatically for Markdown files. Useful when writing
documentation, including this file.

```lua
vim.api.nvim_create_autocmd("FileType", {
    pattern  = "markdown",
    callback = function() vim.opt_local.spell = true end,
})
```

---

## Disabled Builtins

Neovim ships with built-in plugins we either replace or never use. Disabling
them at startup saves a measurable amount of load time.

- `netrw`: the built-in file browser, replaced by neo-tree and oil.nvim.
- `matchparen`: bracket highlighting, handled better by treesitter.
- The rest are legacy file-type handlers (gzip, zip, tar, 2html) and internal
  plugins that serve no purpose in a modern config.

```lua
vim.g.loaded_netrw             = 1
vim.g.loaded_netrwPlugin       = 1
vim.g.loaded_matchparen        = 1
vim.g.loaded_matchit           = 1
vim.g.loaded_logiPat           = 1
vim.g.loaded_rrhelper          = 1
vim.g.loaded_tarPlugin         = 1
vim.g.loaded_gzip              = 1
vim.g.loaded_zipPlugin         = 1
vim.g.loaded_2html_plugin      = 1
vim.g.loaded_shada_plugin      = 1
vim.g.loaded_spellfile_plugin  = 1
vim.g.loaded_tutor_mode_plugin = 1
vim.g.loaded_remote_plugins    = 1
```

---

## GUI (Neovide)

[Neovide](https://neovide.dev) is a GPU-accelerated GUI frontend for Neovim.
These settings only apply when running inside Neovide. The `vim.g.neovide`
guard means they are silently ignored in a terminal.

Font size is controlled by `<D-0>`, `<D-=>`, `<D-->` (the macOS Command key).
The cursor animation adds a smooth trailing effect without being distracting.

```lua
vim.g.gui_font_default_size = 12
vim.g.gui_font_size         = vim.g.gui_font_default_size
vim.g.gui_font_face         = "SFMono Nerd Font"

RefreshGuiFont = function()
    vim.opt.guifont = string.format("%s:h%s", vim.g.gui_font_face, vim.g.gui_font_size)
end
ResizeGuiFont = function(delta)
    vim.g.gui_font_size = vim.g.gui_font_size + delta
    RefreshGuiFont()
end
ResetGuiFont = function()
    vim.g.gui_font_size = vim.g.gui_font_default_size
    RefreshGuiFont()
end
ResetGuiFont()

if vim.g.neovide then
    vim.g.neovide_refresh_rate             = 60
    vim.g.neovide_transparency             = 1.0
    vim.g.neovide_scroll_animation_length  = 0.3
    vim.g.neovide_cursor_animation_length  = 0.05
    vim.g.neovide_cursor_trail_length      = 0.8
    vim.g.neovide_cursor_antialiasing      = false
    vim.g.neovide_cursor_vfx_mode          = "wireframe"
    vim.g.neovide_remember_window_size     = true
    vim.g.neovide_input_use_logo           = true
    vim.g.neovide_input_macos_alt_is_meta  = false
    vim.g.neovide_touch_deadzone           = 0.0
    local o = { noremap = true, silent = true }
    vim.keymap.set({ "n", "i" }, "<D-0>", function() ResetGuiFont() end,    o)
    vim.keymap.set({ "n", "i" }, "<D-=>", function() ResizeGuiFont(1) end,  o)
    vim.keymap.set({ "n", "i" }, "<D-->", function() ResizeGuiFont(-1) end, o)
end
```

---

## Plugin Manager

[lazy.nvim](https://github.com/folke/lazy.nvim) auto-installs itself on first
launch by cloning from GitHub. `--filter=blob:none` makes the clone shallow
(much faster). `--branch=stable` pins to the latest stable release.

`vim.uv` is used rather than the deprecated `vim.loop`. They wrap the same
underlying libuv bindings, but `vim.loop` will be removed in a future Neovim
release.

Plugins are accumulated into a `plugins` table through each section below,
then handed to `require("lazy").setup()` at the end of this file. Each plugin
lives in its own section with its own explanation and configuration.

```lua
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.uv.fs_stat(lazypath) then
    vim.fn.system({
        "git", "clone", "--filter=blob:none",
        "https://github.com/folke/lazy.nvim.git",
        "--branch=stable",
        lazypath,
    })
end
vim.opt.rtp:prepend(lazypath)

local plugins = {}
```

---

## Themes

A collection of quality dark themes. Only one is active at a time; see the
[Theme](#theme) section at the end of this file. Having them all installed
makes it easy to browse and switch live with `<Space>fc` (Telescope colorscheme
picker) without editing the config.

The active theme is **Kanagawa Wave**: a dark navy palette (`#1F1F28`
background) with warm amber keywords and muted green comments. The background
matches a Kitty terminal configured with the Kanagawa theme, so the terminal
padding and the editor blend together.

```lua
vim.list_extend(plugins, {
    "sainnhe/sonokai",
    "tiagovla/tokyodark.nvim",
    "projekt0n/github-nvim-theme",
    "olimorris/onedarkpro.nvim",
    "miikanissi/modus-themes.nvim",
    "rmehri01/onenord.nvim",
    { "catppuccin/nvim", name = "catppuccin" },
    "liuchengxu/space-vim-dark",
    { "ahmedabdulrahman/aylin.vim", branch = "0.5-nvim" },
    "rebelot/kanagawa.nvim",
    "NLKNguyen/papercolor-theme",
    "sainnhe/edge",
    "nyoom-engineering/oxocarbon.nvim",
    "AlexvZyl/nordic.nvim",
    "kaicataldo/material.vim",
    "neanias/everforest-nvim",
})
```

---

## Core Utilities

Libraries used by many other plugins rather than providing features themselves.

- **plenary.nvim**: Lua utility library required by telescope, gitsigns, and
  others. A standard library that the ecosystem shares.
- **nui.nvim**: UI component library used by neo-tree for its menus and inputs.
- **nvim-web-devicons**: file type icons in the tree, bufferline, and
  statusline. Requires a Nerd Font; icons fall back to plain text without one.
- **vim-bbye**: provides `:Bdelete`, a smarter buffer-close. Neovim's built-in
  `:bd` closes the buffer *and* the window. `:Bdelete` closes just the buffer
  and leaves the window open with the next buffer, which is almost always what
  you want.

```lua
vim.list_extend(plugins, {
    "nvim-lua/plenary.nvim",
    "MunifTanjim/nui.nvim",
    "kyazdani42/nvim-web-devicons",
    "moll/vim-bbye",
})
```

---

## File Tree

[neo-tree.nvim](https://github.com/nvim-neo-tree/neo-tree.nvim) is a sidebar
file explorer. It is the primary way to browse and navigate project structure.

`follow_current_file` makes the tree automatically highlight the file currently
open in the editor, so you always know where you are in the project.
`use_libuv_file_watcher` watches the filesystem for external changes (new files
from a terminal, a git checkout, a build process) and updates the tree
automatically.

`hide_dotfiles = false` keeps dotfiles visible. In most projects you need
`.env`, `.gitignore`, `.eslintrc` etc. regularly enough that hiding them is
more inconvenient than seeing them.

`close_if_last_window = true` closes the tree when it is the only window left,
so you are not stuck in an empty layout after closing all editor splits.

```lua
vim.list_extend(plugins, {
    {
        "nvim-neo-tree/neo-tree.nvim",
        branch = "v3.x",
        dependencies = {
            "nvim-lua/plenary.nvim",
            "kyazdani42/nvim-web-devicons",
            "MunifTanjim/nui.nvim",
        },
        config = function()
            require("neo-tree").setup({
                close_if_last_window = true,
                popup_border_style   = "rounded",
                enable_git_status    = true,
                enable_diagnostics   = true,
                window = {
                    position = "left",
                    width    = 35,
                    mappings = {
                        ["<space>"] = "toggle_node",
                        ["<cr>"]    = "open",
                        ["s"]       = "open_vsplit",
                        ["S"]       = "open_split",
                        ["t"]       = "open_tabnew",
                        ["P"]       = { "toggle_preview", config = { use_float = true } },
                        ["C"]       = "close_node",
                        ["z"]       = "close_all_nodes",
                        ["a"]       = "add",
                        ["d"]       = "delete",
                        ["r"]       = "rename",
                        ["y"]       = "copy_to_clipboard",
                        ["x"]       = "cut_to_clipboard",
                        ["p"]       = "paste_from_clipboard",
                        ["q"]       = "close_window",
                        ["R"]       = "refresh",
                        ["?"]       = "show_help",
                    },
                },
                filesystem = {
                    filtered_items = {
                        visible         = false,
                        hide_dotfiles   = false,
                        hide_gitignored = false,
                    },
                    follow_current_file    = { enabled = true },
                    use_libuv_file_watcher = true,
                },
                buffers = {
                    follow_current_file = { enabled = true },
                },
            })
        end,
    },
})
```

---

## File Manager

[oil.nvim](https://github.com/stevearc/oil.nvim) opens the current directory
as an editable buffer. It complements neo-tree rather than replacing it.
Neo-tree gives a persistent sidebar for browsing; oil is faster for bulk
filesystem operations. Rename ten files using standard Vim editing commands,
then save the buffer to apply all changes at once.

```lua
vim.list_extend(plugins, {
    {
        "stevearc/oil.nvim",
        config = function()
            require("oil").setup()
        end,
    },
})
```

---

## Buffer Tabs

[bufferline.nvim](https://github.com/akinsho/bufferline.nvim) renders open
buffers as clickable tabs at the top of the screen. Each tab shows the
filename, a filetype icon, and LSP diagnostic counts from `diagnostics =
"nvim_lsp"`.

`mode = "buffers"` shows one tab per open buffer rather than per vim tab,
which matches how most people think about open files. `sort_by = "id"` keeps
tabs in the order they were opened, which is the most predictable behaviour.

The `offsets` entry reserves the left column for the neo-tree sidebar so
bufferline tabs do not render over it.

```lua
vim.list_extend(plugins, {
    {
        "akinsho/bufferline.nvim",
        config = function()
            require("bufferline").setup({
                options = {
                    mode                         = "buffers",
                    numbers                      = "none",
                    close_command                = "bdelete! %d",
                    right_mouse_command          = "bdelete! %d",
                    left_mouse_command           = "buffer %d",
                    indicator                    = { icon = "▎", style = "icon" },
                    buffer_close_icon            = "",
                    modified_icon                = "●",
                    close_icon                   = "",
                    left_trunc_marker            = "|",
                    right_trunc_marker           = "|",
                    max_name_length              = 18,
                    max_prefix_length            = 15,
                    tab_size                     = 18,
                    diagnostics                  = "nvim_lsp",
                    diagnostics_update_in_insert = false,
                    diagnostics_indicator = function(count, level, diagnostics_dict)
                        local s = " "
                        for e, _ in pairs(diagnostics_dict) do
                            local sym = e == "error" and " "
                                     or (e == "warning" and " " or "")
                            s = s .. sym
                        end
                        return s
                    end,
                    offsets = {
                        { filetype = "neo-tree", text = "Explorer", text_align = "center" },
                    },
                    color_icons             = true,
                    show_buffer_icons       = true,
                    show_buffer_close_icons = true,
                    show_close_icon         = true,
                    show_tab_indicators     = true,
                    separator_style         = "thin",
                    always_show_bufferline  = true,
                    sort_by                 = "id",
                },
            })
        end,
    },
})
```

---

## Status Line

[lualine.nvim](https://github.com/nvim-lualine/lualine.nvim) is the status bar
at the bottom of every window. `theme = "auto"` picks colours based on the
active colorscheme, so it always matches without any manual configuration.

Left side: mode → git branch → diff stats → LSP diagnostics → filename.
Right side: encoding → file format → filetype → cursor position.

```lua
vim.list_extend(plugins, {
    {
        "nvim-lualine/lualine.nvim",
        config = function()
            require("lualine").setup({
                options = {
                    icons_enabled        = true,
                    theme                = "auto",
                    component_separators = { left = "", right = "" },
                    section_separators   = { left = "", right = "" },
                    always_divide_middle = true,
                },
                sections = {
                    lualine_a = { "mode" },
                    lualine_b = { "branch", "diff", "diagnostics" },
                    lualine_c = { "filename" },
                    lualine_x = { "encoding", "fileformat", "filetype" },
                    lualine_z = { "location" },
                },
                inactive_sections = {
                    lualine_a = { "filename" },
                    lualine_x = { "location" },
                },
            })
        end,
    },
})
```

---

## LSP

The Language Server Protocol lets an external program provide Neovim with
language intelligence: completions, go-to-definition, hover documentation,
diagnostics, rename, and code actions. Neovim has a built-in LSP client since
version 0.5. These sections configure which servers to use and how to display
their output.

### Core LSP Plugin

`nvim-lspconfig` provides ready-made configurations for hundreds of language
servers so you do not have to write the server startup commands yourself. From
version 3.0 it uses Neovim's built-in `vim.lsp.config` / `vim.lsp.enable` API
rather than its own setup framework.

```lua
vim.list_extend(plugins, {
    "neovim/nvim-lspconfig",
})
```

### LSP UI: lspsaga

[lspsaga.nvim](https://github.com/nvimdev/lspsaga.nvim) replaces the default
LSP UI with polished floating windows: smooth hover docs, a clean rename
dialog, inline code actions. The lightbulb indicator is disabled because it
appears on every cursor movement and creates constant visual noise.

```lua
vim.list_extend(plugins, {
    {
        "nvimdev/lspsaga.nvim",
        dependencies = {
            "nvim-treesitter/nvim-treesitter",
            "kyazdani42/nvim-web-devicons",
        },
        config = function()
            require("lspsaga").setup({
                ui = {
                    border = "solid",
                    kind   = { Folder = { " " } },
                },
                lightbulb     = { enable = false },
                scroll_preview = {
                    scroll_down = "<C-n>",
                    scroll_up   = "<C-p>",
                },
            })
        end,
    },
})
```

### Definition Viewer: glance

[glance.nvim](https://github.com/dnlhc/glance.nvim) opens definitions,
references, type definitions, and implementations in a floating split rather
than navigating away from your current file. The result list appears on one
side and a preview on the other, so your current context stays visible.

```lua
vim.list_extend(plugins, {
    {
        "dnlhc/glance.nvim",
        config = function()
            local glance  = require("glance")
            local actions = glance.actions
            glance.setup({
                height   = 18,
                zindex   = 45,
                preview_win_opts = { cursorline = true, number = true, wrap = false },
                border           = { enable = false },
                list             = { position = "right", width = 0.33 },
                theme            = { enable = true, mode = "darken" },
                mappings = {
                    list = {
                        ["j"]       = actions.next,
                        ["k"]       = actions.previous,
                        ["<Tab>"]   = actions.next_location,
                        ["<S-Tab>"] = actions.previous_location,
                        ["<C-u>"]   = actions.preview_scroll_win(5),
                        ["<C-d>"]   = actions.preview_scroll_win(-5),
                        ["v"]       = actions.jump_vsplit,
                        ["s"]       = actions.jump_split,
                        ["t"]       = actions.jump_tab,
                        ["<CR>"]    = actions.jump,
                        ["o"]       = actions.jump,
                        ["q"]       = actions.close,
                        ["<Esc>"]   = actions.close,
                    },
                    preview = {
                        ["Q"]       = actions.close,
                        ["<Tab>"]   = actions.next_location,
                        ["<S-Tab>"] = actions.previous_location,
                    },
                },
                folds        = { fold_closed = " ", fold_open = " ", folded = true },
                indent_lines = { enable = true, icon = "│" },
                winbar       = { enable = false },
            })
        end,
    },
})
```

### Per-Project LSP Settings

[nlsp-settings.nvim](https://github.com/tamago324/nlsp-settings.nvim) stores
LSP configuration as JSON files in a `.nlsp-settings/` directory at the
project root. Useful for per-project formatter settings or linter rules
without touching the global config.

```lua
vim.list_extend(plugins, {
    {
        "tamago324/nlsp-settings.nvim",
        config = function()
            require("nlspsettings").setup({
                config_home = vim.fn.stdpath("config") .. "/nlsp-settings",
                local_settings_dir                   = ".nlsp-settings",
                local_settings_root_markers_fallback = { ".git" },
                append_default_schemas               = true,
                loader                               = "json",
            })
        end,
    },
})
```

---

## Snippets

[LuaSnip](https://github.com/L3MON4D3/LuaSnip) is the snippet engine. It
loads snippets from the `snippets/` directory in SnipMate format. The
`_.snippets` file contains global snippets available in every filetype:
currently brackets, Greek letters (useful in mathematical comments), and
blackboard bold symbols.

`filetype_extend("all", { "_" })` makes the global `_` snippets available
everywhere without listing every filetype explicitly.

```lua
vim.list_extend(plugins, {
    {
        "L3MON4D3/LuaSnip",
        build = "make install_jsregexp",
        config = function()
            require("luasnip.loaders.from_snipmate").lazy_load({
                paths = vim.fn.expand("~/.config/nvim/snippets/"),
            })
            require("luasnip").filetype_extend("all", { "_" })
        end,
    },
})
```

---

## Autocompletion

[nvim-cmp](https://github.com/hrsh7th/nvim-cmp) is the completion engine. It
pulls suggestions from multiple sources and presents them in a popup. Sources
in priority order:

1. **otter**: LSP completions from virtual language buffers (see Literate
   Config Support below). Listed first so Lua completions work inside this file.
2. **nvim_lsp**: completions from the language server (most relevant for code).
3. **luasnip**: snippet expansions.
4. **path**: filesystem paths.
5. **buffer**: words from the current buffer (fallback).

`ghost_text = true` shows a faded inline preview of the top completion before
you confirm it, similar to GitHub Copilot ghost text.

Completion is also set up for the command line: `/` and `?` use buffer words
for search, and `:` uses path and cmdline sources.

```lua
vim.list_extend(plugins, {
    "hrsh7th/cmp-nvim-lsp",
    "hrsh7th/cmp-buffer",
    "hrsh7th/cmp-path",
    "hrsh7th/cmp-cmdline",
    {
        "hrsh7th/nvim-cmp",
        dependencies = {
            "hrsh7th/cmp-nvim-lsp",
            "hrsh7th/cmp-buffer",
            "hrsh7th/cmp-path",
            "hrsh7th/cmp-cmdline",
            "L3MON4D3/LuaSnip",
        },
        config = function()
            local cmp     = require("cmp")
            local luasnip = require("luasnip")
            cmp.setup({
                experimental = { ghost_text = true },
                snippet = {
                    expand = function(args)
                        luasnip.lsp_expand(args.body)
                    end,
                },
                mapping = cmp.mapping.preset.insert({
                    ["<C-Space>"] = cmp.mapping.complete(),
                    ["<C-e>"]     = cmp.mapping.abort(),
                    ["<CR>"]      = cmp.mapping.confirm({ select = true }),
                }),
                sources = cmp.config.sources({
                    { name = "otter" },
                    { name = "nvim_lsp" },
                    { name = "luasnip" },
                    { name = "path" },
                }, {
                    { name = "buffer" },
                }),
            })
            cmp.setup.cmdline({ "/", "?" }, {
                mapping = cmp.mapping.preset.cmdline(),
                sources = { { name = "buffer" } },
            })
            cmp.setup.cmdline(":", {
                mapping = cmp.mapping.preset.cmdline(),
                sources = cmp.config.sources(
                    { { name = "path" } },
                    { { name = "cmdline" } }
                ),
                matching = { disallow_symbol_nonprefix_matching = false },
            })
        end,
    },
})
```

---

## Diagnostics Panel

[trouble.nvim](https://github.com/folke/trouble.nvim) provides a panel at the
bottom of the screen listing every diagnostic across the workspace: errors,
warnings, hints. It is much more usable than the built-in location list when
navigating errors spread across multiple files.

```lua
vim.list_extend(plugins, {
    {
        "folke/trouble.nvim",
        dependencies = { "kyazdani42/nvim-web-devicons" },
        config = function()
            require("trouble").setup({
                position    = "bottom",
                height      = 10,
                icons       = true,
                mode        = "workspace_diagnostics",
                fold_open   = "",
                fold_closed = "",
                group       = true,
                padding     = true,
                action_keys = {
                    close          = "q",
                    cancel         = "<esc>",
                    refresh        = "r",
                    jump           = { "<cr>", "<tab>" },
                    open_split     = { "<c-x>" },
                    open_vsplit    = { "<c-v>" },
                    open_tab       = { "<c-t>" },
                    jump_close     = { "o" },
                    toggle_mode    = "m",
                    toggle_preview = "P",
                    hover          = "K",
                    preview        = "p",
                    close_folds    = { "zM", "zm" },
                    open_folds     = { "zR", "zr" },
                    toggle_fold    = { "zA", "za" },
                    previous       = "k",
                    next           = "j",
                },
                indent_lines = true,
                auto_open    = false,
                auto_close   = false,
                auto_preview = true,
                auto_fold    = false,
                auto_jump    = { "lsp_definitions" },
                signs = {
                    error       = "",
                    warning     = "",
                    hint        = "",
                    information = "",
                    other       = "󰮍",
                },
                use_diagnostic_signs = false,
            })
        end,
    },
})
```

---

## Syntax Highlighting

[nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter) parses
source code into a concrete syntax tree that Neovim uses for accurate, fast
highlighting. It is dramatically better than the old regex-based approach and
enables many other plugins (folding, text objects, etc.) to work correctly.

`branch = "main"` uses the new treesitter API. The old
`require("nvim-treesitter.configs").setup()` call no longer works on the main
branch. The new setup is a simple `require("nvim-treesitter").setup()`.

Parsers for `c`, `cpp`, and `python` cover the languages used in this config.
`lua` is added because the config itself is Lua and having the parser improves
the editing experience here in the literate document.

Treesitter compiles parsers from C source on install, which requires `gcc` and
`make` (installed via the Prerequisites block above).

```lua
vim.list_extend(plugins, {
    {
        "nvim-treesitter/nvim-treesitter",
        branch = "main",
        build  = ":TSUpdate",
        config = function()
            require("nvim-treesitter").setup({
                ensure_installed = { "c", "cpp", "python", "lua" },
                highlight        = { enable = true },
            })
        end,
    },
})
```

---

## Git Integration

[gitsigns.nvim](https://github.com/lewis6991/gitsigns.nvim) shows git diff
markers in the sign column: `│` for added and changed lines, `_` for deleted
lines. It also provides hunk-level operations: stage a single block of changes,
reset it, preview the diff, or blame a line, all without leaving Neovim.

`current_line_blame = false` keeps inline blame off by default to avoid
cluttering the screen. Toggle it for a session with
`:Gitsigns toggle_current_line_blame` when you need it.

```lua
vim.list_extend(plugins, {
    {
        "lewis6991/gitsigns.nvim",
        config = function()
            require("gitsigns").setup({
                signs = {
                    add          = { text = "│" },
                    change       = { text = "│" },
                    delete       = { text = "_" },
                    topdelete    = { text = "‾" },
                    changedelete = { text = "~" },
                },
                signcolumn         = true,
                current_line_blame = false,
                current_line_blame_opts = {
                    virt_text     = true,
                    virt_text_pos = "eol",
                    delay         = 1000,
                },
                preview_config = {
                    border   = "single",
                    style    = "minimal",
                    relative = "cursor",
                    row = 0, col = 1,
                },
            })
        end,
    },
})
```

---

## Terminal

[toggleterm.nvim](https://github.com/akinsho/toggleterm.nvim) embeds terminal
instances inside Neovim. The default direction is horizontal (a split at the
bottom). Multiple independent instances can be opened with `<Space>tn`.

The floating terminal (`<Space>tf`) is useful for quick one-off commands
without disrupting the editor layout: run a command, close it, continue.

Size 30 means the horizontal split occupies 30 rows, enough terminal space
without dominating the screen.

```lua
vim.list_extend(plugins, {
    {
        "akinsho/toggleterm.nvim",
        config = function()
            require("toggleterm").setup({
                size           = 30,
                open_mapping   = "<f5>",
                hide_numbers   = true,
                shade_terminals = true,
                shading_factor  = "1",
                start_in_insert = true,
                persist_size    = true,
                persist_mode    = true,
                direction       = "horizontal",
                close_on_exit   = true,
                shell           = vim.o.shell,
                auto_scroll     = true,
                float_opts = {
                    border   = "single",
                    width    = 140,
                    height   = 50,
                    winblend = 1,
                },
            })
        end,
    },
})
```

---

## Fuzzy Finder

[telescope.nvim](https://github.com/nvim-telescope/telescope.nvim) is a fuzzy
finder with a live preview pane. It handles file search, text search across the
project (via ripgrep), buffer switching, colorscheme browsing, register
inspection, and help tag search.

`live_grep` (text search across the project) requires ripgrep. `fd-find`
speeds up file search. On Ubuntu the binary is called `fdfind`.

```sh install
PKG ripgrep
# fd has different package names — Ubuntu calls it fd-find and installs as fdfind
if command -v apt-get &>/dev/null; then
    PKG fd-find
    mkdir -p ~/.local/bin
    ln -sf "$(which fdfind)" ~/.local/bin/fd 2>/dev/null || true
else
    PKG fd
fi
```

```lua
vim.list_extend(plugins, {
    {
        "nvim-telescope/telescope.nvim",
        dependencies = { "nvim-lua/plenary.nvim" },
        config = function()
            require("telescope").setup({
                defaults = {
                    mappings = {
                        i = { ["<C-h>"] = "which_key" },
                    },
                },
            })
        end,
    },
})
```

---

## Literate Config Support

Two plugins that directly improve the experience of editing this file.

### otter.nvim

[otter.nvim](https://github.com/jmbuhr/otter.nvim) injects LSP intelligence
into code blocks inside markdown files. When your cursor is inside a ` ```lua `
block in this README, otter creates a hidden Lua buffer behind the scenes. The
language server attaches to that buffer, giving you full completions,
diagnostics, hover docs, and go-to-definition, exactly as if you were editing
a `.lua` file directly.

This covers the main pain point of the literate config workflow: editing Lua
code inside a Markdown file without losing language intelligence.

```lua
vim.list_extend(plugins, {
    {
        "jmbuhr/otter.nvim",
        dependencies = { "nvim-treesitter/nvim-treesitter" },
        config = function()
            require("otter").setup({
                lsp = {
                    hover = { border = "rounded" },
                },
                buffers = { set_filetype = true },
            })
            vim.api.nvim_create_autocmd("FileType", {
                pattern  = "markdown",
                callback = function()
                    require("otter").activate({ "lua" }, true, true, nil)
                end,
            })
        end,
    },
})
```

### render-markdown.nvim

[render-markdown.nvim](https://github.com/MeanderingProgrammer/render-markdown.nvim)
renders markdown headings, bold, italic, code blocks, and tables with visual
formatting directly in the buffer. It only renders what is visible on screen,
so it stays fast even on large files like this README.

Toggle rendering with `:RenderMarkdown toggle` if it gets in the way during
heavy editing.

```lua
vim.list_extend(plugins, {
    {
        "MeanderingProgrammer/render-markdown.nvim",
        dependencies = {
            "nvim-treesitter/nvim-treesitter",
            "kyazdani42/nvim-web-devicons",
        },
        config = function()
            require("render-markdown").setup({
                file_types = { "markdown" },
            })
        end,
    },
})
```

---

## Key Hints

[which-key.nvim](https://github.com/folke/which-key.nvim) shows a popup after
you press the leader key, listing the available next keys with descriptions.
After `delay` milliseconds it appears with a grouped, labelled menu. Press a
letter to continue the sequence, or `Esc` to cancel.

`icons.mappings = false` disables the icon column next to each binding, keeping
the popup clean without requiring icon-specific fonts.

```lua
vim.list_extend(plugins, {
    {
        "folke/which-key.nvim",
        event  = "VeryLazy",
        config = function()
            local wk = require("which-key")
            wk.setup({
                delay = 300,
                icons = { mappings = false },
                win   = { border = "rounded" },
            })
            wk.add({
                { "<leader>e",  group = "File Tree" },
                { "<leader>f",  group = "Find (Telescope)" },
                { "<leader>b",  group = "Buffers" },
                { "<leader>w",  group = "Windows / Splits" },
                { "<leader>s",  group = "Search" },
                { "<leader>t",  group = "Terminal" },
                { "<leader>l",  group = "LSP / Code" },
                { "<leader>g",  group = "Go To" },
                { "<leader>d",  group = "Diagnostics" },
                { "<leader>h",  group = "Git Hunks" },
                { "<leader>v",  group = "Visual / Theme" },
                { "<leader>p",  group = "Plugins" },
                { "<leader>W",  group = "Workspace (LSP)" },
                { "<leader>ee", desc = "Toggle file tree" },
                { "<leader>ef", desc = "Reveal current file in tree" },
                { "<leader>er", desc = "Refresh file tree" },
                { "<leader>ff", desc = "Find file by name" },
                { "<leader>fg", desc = "Find text in project" },
                { "<leader>fb", desc = "Find open buffer" },
                { "<leader>fc", desc = "Find / preview colorscheme" },
                { "<leader>fr", desc = "Find in registers" },
                { "<leader>fh", desc = "Find help tag" },
                { "<leader>bn", desc = "Next buffer" },
                { "<leader>bp", desc = "Previous buffer" },
                { "<leader>bd", desc = "Close buffer" },
                { "<leader>bb", desc = "Browse open buffers" },
                { "<leader>ws", desc = "Split horizontal" },
                { "<leader>wv", desc = "Split vertical" },
                { "<leader>wx", desc = "Close current window" },
                { "<leader>wo", desc = "Close all other windows" },
                { "<leader>ss", desc = "Search in file" },
                { "<leader>sw", desc = "Search exact whole word" },
                { "<leader>th", desc = "Terminal horizontal" },
                { "<leader>tv", desc = "Terminal vertical" },
                { "<leader>tf", desc = "Terminal floating" },
                { "<leader>tt", desc = "Terminal new tab" },
                { "<leader>tn", desc = "New terminal instance" },
                { "<leader>lk", desc = "Hover docs" },
                { "<leader>ld", desc = "Preview definition" },
                { "<leader>lr", desc = "Rename symbol" },
                { "<leader>lh", desc = "Signature / parameters" },
                { "<leader>lf", desc = "Format file" },
                { "<leader>la", desc = "Code actions" },
                { "<leader>lq", desc = "Diagnostics to location list" },
                { "<leader>gd", desc = "Definitions" },
                { "<leader>gr", desc = "References" },
                { "<leader>gt", desc = "Type definitions" },
                { "<leader>gi", desc = "Implementations" },
                { "<leader>gn", desc = "Next diagnostic" },
                { "<leader>gp", desc = "Previous diagnostic" },
                { "<leader>dd", desc = "Line diagnostics" },
                { "<leader>ds", desc = "Symbol diagnostics" },
                { "<leader>dl", desc = "Toggle Trouble panel" },
                { "<leader>df", desc = "Focus Trouble panel" },
                { "<leader>hn", desc = "Next hunk" },
                { "<leader>hN", desc = "Previous hunk" },
                { "<leader>hc", desc = "Preview hunk diff" },
                { "<leader>hs", desc = "Stage hunk" },
                { "<leader>hu", desc = "Unstage hunk" },
                { "<leader>hr", desc = "Reset hunk" },
                { "<leader>hR", desc = "Reset entire file" },
                { "<leader>hl", desc = "Blame current line" },
                { "<leader>hd", desc = "Diff this file" },
                { "<leader>vd", desc = "Dark background" },
                { "<leader>vl", desc = "Light background" },
                { "<leader>pi", desc = "Install / update plugins" },
                { "<leader>pc", desc = "Clean unused plugins" },
                { "<leader>Wa", desc = "Add workspace folder" },
                { "<leader>Wr", desc = "Remove workspace folder" },
                { "<leader>Wl", desc = "List workspace folders" },
            })
        end,
    },
})
```

---

## Finalize Plugin Setup

All plugin specs have been accumulated into the `plugins` table through the
sections above. They are now handed to lazy.nvim to manage installation and
loading.

```lua
require("lazy").setup(plugins)
```

---

## LSP Server Configuration

Language server configuration using Neovim 0.11+'s built-in API. This runs
after lazy.setup so nvim-lspconfig is in the runtime path.

Two servers are configured:

- **clangd** for C and C++. Developed by the LLVM project; the fastest and
  most complete C++ LSP available. `-header-insertion=never` stops clangd
  from automatically adding `#include` directives, which interferes with
  intentional include management.
- **pylsp** for Python, the Python Language Server.

Both must be installed on your system separately:

```sh install
# Package names differ per distro
if command -v apt-get &>/dev/null; then
    PKG clangd python3-pylsp
elif command -v pacman &>/dev/null; then
    PKG clang python-pylsp
else
    PKG llvm python-lsp-server
fi
```

```lua
vim.lsp.config("clangd", {
    cmd = { "clangd", "-header-insertion=never" },
})
vim.lsp.config("pylsp", {})
vim.lsp.enable({ "clangd", "pylsp" })
```

---

## Keymaps

All keymaps in one place, defined after plugins load. Plugin modules are
required inside function bodies (lazy-required) so these definitions are safe
even before the plugins are loaded.

`jk` as Escape keeps your fingers on the home row and avoids the reach to
`Esc`. The sequence `jk` does not appear in normal English typing, so it never
interferes with writing prose.

`Ctrl+h/j/k/l` for moving between splits mirrors the hjkl movement keys and
feels natural after a short adjustment period.

```lua
-- ── Mode escaping ─────────────────────────────────────────────────────────
vim.keymap.set("i", "jk",   "<esc>",       { silent = true })
vim.keymap.set("t", "jk",   "<C-\\><C-n>", { silent = true })
vim.keymap.set("t", "<C-g>","<C-\\><C-n>", { silent = true })

-- C++ scope resolution operator in insert mode
vim.api.nvim_create_autocmd("FileType", {
    pattern  = { "cpp" },
    callback = function()
        vim.schedule(function()
            vim.keymap.set("i", "<C-;>", "::", { buffer = true })
        end)
    end,
})

-- ── File tree ─────────────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>ee", ":Neotree toggle<cr>", { silent = true })
vim.keymap.set("n", "<leader>ef", ":Neotree reveal<cr>", { silent = true })
vim.keymap.set("n", "<leader>er", function()
    require("neo-tree.command").execute({ action = "refresh" })
end, { silent = true })

-- ── Fuzzy find ────────────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>ff", function() require("telescope.builtin").find_files() end)
vim.keymap.set("n", "<leader>fg", function() require("telescope.builtin").live_grep() end)
vim.keymap.set("n", "<leader>fb", function() require("telescope.builtin").buffers() end)
vim.keymap.set("n", "<leader>fc", function() require("telescope.builtin").colorscheme() end)
vim.keymap.set("n", "<leader>fr", function() require("telescope.builtin").registers() end)
vim.keymap.set("n", "<leader>fh", function() require("telescope.builtin").help_tags() end)

-- ── Buffers ───────────────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>bn", ":bn<cr>")
vim.keymap.set("n", "<leader>bp", ":bp<cr>")
vim.keymap.set("n", "<leader>bd", ":Bdelete<cr>")
vim.keymap.set("n", "<leader>bb", function() require("telescope.builtin").buffers() end)

-- ── Windows / splits ──────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>ws", ":sp<cr>")
vim.keymap.set("n", "<leader>wv", ":vs<cr>")
vim.keymap.set("n", "<leader>wx", ":q<cr>")
vim.keymap.set("n", "<leader>wo", "<C-w>o")
vim.keymap.set("n", "<C-h>", "<C-w>h")
vim.keymap.set("n", "<C-j>", "<C-w>j")
vim.keymap.set("n", "<C-k>", "<C-w>k")
vim.keymap.set("n", "<C-l>", "<C-w>l")
vim.keymap.set("n", "<A-h>", "<C-w><", { silent = true })
vim.keymap.set("n", "<A-l>", "<C-w>>", { silent = true })
vim.keymap.set("n", "<A-j>", "<C-w>-", { silent = true })
vim.keymap.set("n", "<A-k>", "<C-w>+", { silent = true })

-- ── Search ────────────────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>ss", "/")
vim.keymap.set("n", "<leader>sw", "/\\<lt>\\><left><left>")

-- ── File manager (oil) ────────────────────────────────────────────────────
vim.keymap.set("n", "<F3>",       ":Oil<cr>")
vim.keymap.set("n", "<leader>ft", ":Oil<cr>")

-- ── Terminal ──────────────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>th", ":ToggleTerm direction=horizontal<cr>")
vim.keymap.set("n", "<leader>tv", ":ToggleTerm direction=vertical<cr>")
vim.keymap.set("n", "<leader>tf", ":ToggleTerm direction=float<cr>")
vim.keymap.set("n", "<leader>tt", ":ToggleTerm direction=tab<cr>")
vim.keymap.set("n", "<leader>tn", function()
    require("toggleterm.terminal").Terminal:new():toggle()
end)

-- ── LSP ───────────────────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>lk", ":Lspsaga hover_doc<cr>")
vim.keymap.set("n", "<leader>ld", ":Lspsaga preview_definition<cr>")
vim.keymap.set("n", "<leader>lr", ":Lspsaga rename<cr>")
vim.keymap.set("n", "<leader>lh", function() vim.lsp.buf.signature_help() end)
vim.keymap.set("n", "<leader>lf", function() vim.lsp.buf.format({ async = true }) end)
vim.keymap.set("n", "<leader>la", ":Lspsaga code_action<cr>")
vim.keymap.set("n", "<leader>lq", function() vim.diagnostic.setloclist() end)

-- ── Go to ─────────────────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>gd", ":Glance definitions<CR>")
vim.keymap.set("n", "<leader>gr", ":Glance references<CR>")
vim.keymap.set("n", "<leader>gt", ":Glance type_definitions<CR>")
vim.keymap.set("n", "<leader>gi", ":Glance implementations<CR>")
vim.keymap.set("n", "<leader>gn", ":Lspsaga diagnostic_jump_next<cr>")
vim.keymap.set("n", "<leader>gp", ":Lspsaga diagnostic_jump_prev<cr>")

-- ── Diagnostics ───────────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>dd", ":Lspsaga show_line_diagnostics<cr>")
vim.keymap.set("n", "<leader>ds", ":Lspsaga show_cursor_diagnostics<cr>")
vim.keymap.set("n", "<leader>dl", ":TroubleToggle<cr>")
vim.keymap.set("n", "<leader>df", ":Trouble<cr>")

-- ── Git ───────────────────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>hn", ":Gitsigns next_hunk<cr>")
vim.keymap.set("n", "<leader>hN", ":Gitsigns prev_hunk<cr>")
vim.keymap.set("n", "<leader>hc", ":Gitsigns preview_hunk<cr>")
vim.keymap.set("n", "<leader>hs", ":Gitsigns stage_hunk<cr>")
vim.keymap.set("n", "<leader>hu", ":Gitsigns undo_stage_hunk<cr>")
vim.keymap.set("n", "<leader>hr", ":Gitsigns reset_hunk<cr>")
vim.keymap.set("n", "<leader>hR", ":Gitsigns reset_buffer<cr>")
vim.keymap.set("n", "<leader>hl", ":Gitsigns blame_line<cr>")
vim.keymap.set("n", "<leader>hd", ":Gitsigns diffthis<cr>")

-- ── Snippets ──────────────────────────────────────────────────────────────
vim.keymap.set("i", "<C-o>", function() require("luasnip").expand() end, { silent = true })
vim.keymap.set({ "i", "s" }, "<C-l>", function() require("luasnip").jump(1) end, { silent = true })
vim.keymap.set({ "i", "s" }, "<C-k>", function()
    local ls = require("luasnip")
    if ls.choice_active() then ls.change_choice(1) end
end, { silent = true })

-- ── Theme toggles ─────────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>vd", function() vim.cmd("set background=dark") end)
vim.keymap.set("n", "<leader>vl", function() vim.cmd("set background=light") end)

-- ── Plugins ───────────────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>pi", ":Lazy sync<cr>")
vim.keymap.set("n", "<leader>pc", ":Lazy clean<cr>")

-- ── Workspace ─────────────────────────────────────────────────────────────
vim.keymap.set("n", "<leader>Wa", function() vim.lsp.buf.add_workspace_folder() end)
vim.keymap.set("n", "<leader>Wr", function() vim.lsp.buf.remove_workspace_folder() end)
vim.keymap.set("n", "<leader>Wl", function()
    print(vim.inspect(vim.lsp.buf.list_workspace_folders()))
end)
```

---

## Theme

Kanagawa Wave is the active theme. Its background (`#1F1F28`) matches a Kitty
terminal configured with the Kanagawa colour palette, so the terminal padding
and the editor background blend together.

To switch themes: change the `colorscheme` line and save. Or use `<Space>fc`
to browse all installed themes with live preview, then update this line once
you have decided.

```lua
require("kanagawa").setup({
    theme = "wave",  -- "wave" | "dragon" | "lotus"
})
vim.cmd("colorscheme kanagawa-wave")
vim.cmd("set background=dark")
```
