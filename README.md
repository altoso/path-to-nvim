# Path to Cool Vim

A step-by-step guide to turning a bare Linux terminal into a modern Neovim setup with TypeScript LSP, ESLint linting, autocompletion, fuzzy finding, and a clean UI.

## Prerequisites

- Linux (Debian/Ubuntu-based, including WSL2)
- Node.js 18+ and npm (required for the TypeScript and ESLint language servers)
- git, curl, tar (standard on most distros)
- A terminal with true color support

Check you have them:

```bash
node --version   # should be >= 18
npm --version
git --version
curl --version
```

---

## Step 1 — Install Neovim 0.12+

The distro package is usually too old. Install the latest stable release from GitHub into `~/.local`:

```bash
curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim-linux-x86_64.tar.gz
tar -C ~/.local -xzf nvim-linux-x86_64.tar.gz
ln -sf ~/.local/nvim-linux-x86_64/bin/nvim ~/.local/bin/nvim
rm nvim-linux-x86_64.tar.gz
```

Add `~/.local/bin` to your PATH permanently:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.profile
source ~/.bashrc
```

Verify:

```bash
nvim --version   # should show NVIM v0.12.x
```

---

## Step 2 — Create the config directory structure

```bash
mkdir -p ~/.config/nvim/lua/plugins
```

---

## Step 3 — Write `init.lua`

Create `~/.config/nvim/init.lua`:

```lua
-- Bootstrap lazy.nvim
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({
    "git", "clone", "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable", lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)

-- Leader key
vim.g.mapleader = " "
vim.g.maplocalleader = " "

-- Basic options
vim.opt.number = true
vim.opt.relativenumber = true
vim.opt.tabstop = 2
vim.opt.shiftwidth = 2
vim.opt.expandtab = true
vim.opt.smartindent = true
vim.opt.wrap = false
vim.opt.termguicolors = true
vim.opt.signcolumn = "yes"
vim.opt.updatetime = 250
vim.opt.timeoutlen = 300
vim.opt.splitright = true
vim.opt.splitbelow = true
vim.opt.ignorecase = true
vim.opt.smartcase = true
vim.opt.scrolloff = 8
vim.opt.undofile = true
vim.opt.clipboard = "unnamedplus"

require("lazy").setup("plugins", {
  change_detection = { notify = false },
})
```

---

## Step 4 — Write the plugins spec

Create `~/.config/nvim/lua/plugins/init.lua`:

```lua
return {
  -- Colorscheme
  {
    "catppuccin/nvim",
    name = "catppuccin",
    priority = 1000,
    config = function()
      require("catppuccin").setup({ flavour = "mocha" })
      vim.cmd.colorscheme("catppuccin")
    end,
  },

  -- Status line
  {
    "nvim-lualine/lualine.nvim",
    dependencies = { "nvim-tree/nvim-web-devicons" },
    opts = {
      options = { theme = "catppuccin" },
    },
  },

  -- File explorer
  {
    "nvim-tree/nvim-tree.lua",
    dependencies = { "nvim-tree/nvim-web-devicons" },
    keys = { { "<leader>e", "<cmd>NvimTreeToggle<cr>", desc = "File Explorer" } },
    opts = {},
  },

  -- Fuzzy finder
  {
    "nvim-telescope/telescope.nvim",
    dependencies = { "nvim-lua/plenary.nvim" },
    keys = {
      { "<leader>ff", "<cmd>Telescope find_files<cr>", desc = "Find Files" },
      { "<leader>fg", "<cmd>Telescope live_grep<cr>",  desc = "Live Grep" },
      { "<leader>fb", "<cmd>Telescope buffers<cr>",    desc = "Buffers" },
      { "<leader>fh", "<cmd>Telescope help_tags<cr>",  desc = "Help" },
    },
  },

  -- Treesitter (syntax + indentation)
  {
    "nvim-treesitter/nvim-treesitter",
    lazy = false,
    build = ":TSUpdate",
    config = function()
      require("nvim-treesitter").setup()
      require("nvim-treesitter").install({
        "typescript", "tsx", "javascript", "json",
        "lua", "html", "css", "markdown", "bash",
      })
    end,
  },

  -- Mason (LSP server installer)
  {
    "mason-org/mason.nvim",
    build = ":MasonUpdate",
    opts = {},
  },

  -- mason-lspconfig (auto-enable installed servers)
  {
    "mason-org/mason-lspconfig.nvim",
    dependencies = {
      "mason-org/mason.nvim",
      "neovim/nvim-lspconfig",
    },
    opts = {
      ensure_installed = { "ts_ls", "eslint" },
    },
  },

  -- LSP config (Neovim 0.11+ new API)
  {
    "neovim/nvim-lspconfig",
    dependencies = { "hrsh7th/cmp-nvim-lsp" },
    config = function()
      local capabilities = require("cmp_nvim_lsp").default_capabilities()

      -- TypeScript LSP
      vim.lsp.config("ts_ls", { capabilities = capabilities })

      -- ESLint LSP: auto-fix on save
      vim.lsp.config("eslint", {
        capabilities = capabilities,
        on_attach = function(_, bufnr)
          vim.api.nvim_create_autocmd("BufWritePre", {
            buffer = bufnr,
            command = "EslintFixAll",
          })
        end,
      })

      vim.lsp.enable({ "ts_ls", "eslint" })

      -- LSP keymaps on attach
      vim.api.nvim_create_autocmd("LspAttach", {
        callback = function(ev)
          local opts = { buffer = ev.buf }
          vim.keymap.set("n", "gd",         vim.lsp.buf.definition,     opts)
          vim.keymap.set("n", "gD",         vim.lsp.buf.declaration,    opts)
          vim.keymap.set("n", "gr",         vim.lsp.buf.references,     opts)
          vim.keymap.set("n", "gi",         vim.lsp.buf.implementation, opts)
          vim.keymap.set("n", "K",          vim.lsp.buf.hover,          opts)
          vim.keymap.set("n", "<leader>rn", vim.lsp.buf.rename,         opts)
          vim.keymap.set("n", "<leader>ca", vim.lsp.buf.code_action,    opts)
          vim.keymap.set("n", "<leader>f",  vim.lsp.buf.format,         opts)
          vim.keymap.set("n", "[d",         vim.diagnostic.goto_prev,   opts)
          vim.keymap.set("n", "]d",         vim.diagnostic.goto_next,   opts)
        end,
      })
    end,
  },

  -- Autocompletion
  {
    "hrsh7th/nvim-cmp",
    dependencies = {
      "hrsh7th/cmp-nvim-lsp",
      "hrsh7th/cmp-buffer",
      "hrsh7th/cmp-path",
      "L3MON4D3/LuaSnip",
      "saadparwaiz1/cmp_luasnip",
      "rafamadriz/friendly-snippets",
    },
    config = function()
      local cmp = require("cmp")
      local luasnip = require("luasnip")
      require("luasnip.loaders.from_vscode").lazy_load()

      cmp.setup({
        snippet = {
          expand = function(args) luasnip.lsp_expand(args.body) end,
        },
        mapping = cmp.mapping.preset.insert({
          ["<C-b>"]     = cmp.mapping.scroll_docs(-4),
          ["<C-f>"]     = cmp.mapping.scroll_docs(4),
          ["<C-Space>"] = cmp.mapping.complete(),
          ["<C-e>"]     = cmp.mapping.abort(),
          ["<CR>"]      = cmp.mapping.confirm({ select = true }),
          ["<Tab>"] = cmp.mapping(function(fallback)
            if cmp.visible() then cmp.select_next_item()
            elseif luasnip.expand_or_jumpable() then luasnip.expand_or_jump()
            else fallback() end
          end, { "i", "s" }),
          ["<S-Tab>"] = cmp.mapping(function(fallback)
            if cmp.visible() then cmp.select_prev_item()
            elseif luasnip.jumpable(-1) then luasnip.jump(-1)
            else fallback() end
          end, { "i", "s" }),
        }),
        sources = cmp.config.sources({
          { name = "nvim_lsp" },
          { name = "luasnip" },
          { name = "buffer" },
          { name = "path" },
        }),
      })
    end,
  },

  -- Auto pairs
  {
    "windwp/nvim-autopairs",
    event = "InsertEnter",
    opts = {},
  },

  -- Keybinding hints
  {
    "folke/which-key.nvim",
    event = "VeryLazy",
    opts = {},
  },

  -- Git signs in gutter
  {
    "lewis6991/gitsigns.nvim",
    opts = {},
  },

  -- Comment toggling (gcc / gc<motion>)
  {
    "numToStr/Comment.nvim",
    opts = {},
  },

  -- Indentation guides
  {
    "lukas-reineke/indent-blankline.nvim",
    main = "ibl",
    opts = {},
  },

  -- Diagnostics panel
  {
    "folke/trouble.nvim",
    dependencies = { "nvim-tree/nvim-web-devicons" },
    keys = {
      { "<leader>xx", "<cmd>Trouble diagnostics toggle<cr>",              desc = "Diagnostics" },
      { "<leader>xd", "<cmd>Trouble diagnostics toggle filter.buf=0<cr>", desc = "Buffer Diagnostics" },
    },
    opts = {},
  },

  -- Surround text objects (ys / ds / cs)
  {
    "kylechui/nvim-surround",
    event = "VeryLazy",
    opts = {},
  },
}
```

---

## Step 5 — Install plugins headlessly

This clones all plugins without opening the UI:

```bash
nvim --headless "+Lazy! sync" +qa
```

The first run takes about 30–60 seconds depending on your connection.

---

## Step 6 — Install the LSP servers

Mason handles downloading and installing the language servers:

```bash
nvim --headless "+MasonInstall typescript-language-server eslint-lsp" +qa
```

Verify they landed:

```bash
ls ~/.local/share/nvim/mason/bin/
# should list: typescript-language-server  vscode-eslint-language-server
```

---

## Step 7 — Launch and verify

```bash
nvim
```

On first open, treesitter will compile parsers in the background (you may see progress messages). Once that settles:

- Open any `.ts` or `.tsx` file — you should get syntax highlighting, hover docs (`K`), and ESLint diagnostics in the gutter within a couple of seconds.
- Press `<Space>` and wait a moment — which-key will show available keybindings.

---

## Keybinding reference

| Key | Action |
|-----|--------|
| `<leader>e` | Toggle file explorer |
| `<leader>ff` | Find files |
| `<leader>fg` | Live grep |
| `<leader>fb` | Buffers |
| `gd` | Go to definition |
| `gD` | Go to declaration |
| `gr` | References |
| `gi` | Implementation |
| `K` | Hover docs |
| `<leader>rn` | Rename symbol |
| `<leader>ca` | Code action |
| `<leader>f` | Format file |
| `[d` / `]d` | Prev / next diagnostic |
| `<leader>xx` | Toggle diagnostics panel |
| `<leader>xd` | Buffer diagnostics |
| `gcc` | Toggle comment (line) |
| `gc<motion>` | Toggle comment (motion) |
| `ys<motion><char>` | Surround with char |
| `ds<char>` | Delete surrounding char |
| `cs<old><new>` | Change surrounding char |

---

## Updating

- **Plugins:** `:Lazy update` inside Neovim
- **LSP servers:** `:Mason` → `U` to update all
- **Treesitter parsers:** `:TSUpdate`
- **Neovim itself:** re-run Step 1 (the symlink at `~/.local/bin/nvim` will pick up the new version automatically)
