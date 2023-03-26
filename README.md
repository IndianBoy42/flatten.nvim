# flatten.nvim

Flatten allows you to open files from a neovim terminal buffer in your current neovim instance instead of a nested one.

## Features

- [x] Open files from terminal buffers without creating a nested session
- [x] Allow blocking for git commits
- [x] Configuration
  - [x] Callbacks for user-specific workflows
  - [x] Open in vsplit, split, tab, or current window
- [x] Pipe from terminal into a new Neovim buffer ([demo](https://user-images.githubusercontent.com/38540736/225779817-ed7efea8-9108-4f28-983f-1a889d32826f.mp4))
- [x] Setting to force blocking from the commandline, regardless of filetype

## Plans and Ideas

Ideas:

- [ ] Multi-screen support
  - [ ] Move buffers between Neovim instances in separate windows
  - [ ] Single cursor between Neovim instances in separate windows

If you have an idea or feature request, open an issue with the `enhancement` tag!

## Demo

https://user-images.githubusercontent.com/38540736/224443095-91450818-f298-4e08-a951-ee3fcc607330.mp4

Config for demo [here](#advanced-configuration) (autodelete gitcommit on write and toggling terminal are not defaults)

## Installation[^1]

With `lazy.nvim`:

```lua

{
    'willothy/flatten.nvim',
    config = true,
    -- or pass configuration with
    -- opts = {  }
    -- Ensure that it runs first to minimize delay when opening file from terminal
    lazy = false, priority = 1001,
}

```

To avoid loading plugins in guest sessions you can use the following in your config:

```lua
-- If opening from inside neovim terminal then do not load all the other plugins
if os.getenv("NVIM") ~= nil then
    require('lazy').setup {
        {'willothy/flatten.nvim', config = true },
    }
    return
end

-- Otherwise proceed as normal
require('lazy').setup( --[[ your normal config ]] )
```

## Usage

```zsh
# open files normally
nvim file1 file2

# force blocking for a file
nvim --cmd 'let g:flatten_wait=1' file1

# enable blocking for $VISUAL
# allows edit-exec
# in your .bashrc, .zshrc, etc.
export VISUAL="nvim --cmd 'let g:flatten_wait=1'"
```

## Configuration

### Defaults

Flatten comes with the following defaults:

```lua
{
    callbacks = {
        -- Called when a request to edit file(s) is received
        pre_open = function() end,
        -- Called after a file is opened
        -- Passed the buf id, win id, and filetype of the new window
        post_open = function(bufnr, winnr, filetype) end,
        -- Called when a file is open in blocking mode, after it's done blocking
        -- (after bufdelete, bufunload, or quitpre for the blocking buffer)
        block_end = function() end,
    },
    -- <String, Bool> dictionary of filetypes that should be blocking
    block_for = {
        gitcommit = true
    },
    -- Window options
    window = {
        -- Options:
        -- current        -> open in current window (default)
        -- alternate      -> open in alternate window (recommended)
        -- tab            -> open in new tab
        -- split          -> open in split
        -- vsplit         -> open in vsplit
        -- func(new_bufs) -> only open the files, allowing you to handle window opening yourself.
        -- Argument is an array of buffer numbers representing the newly opened files.
        open = "current",
        -- Affects which file gets focused when opening multiple at once
        -- Options:
        -- "first"        -> open first file of new files (default)
        -- "last"         -> open last file of new files
        focus = "first"
    }
}
```

### Advanced configuration

If you use a toggleable terminal and don't want the new buffer(s) to be opened in your current window, you can use the `alternate` mode instead of `current` to open in your last window. With this method, the terminal doesn't need to be closed and re-opened as it did with the [old example config](https://github.com/willothy/flatten.nvim/blob/c986f98bc1d1e2365dfb2e97dda58ca5d0ae24ae/README.md).

The only reason 'alternate' isn't the default is to avoid breaking people's configs. It may become the default at some point if that's something that people ask for (e.g., open an issue if you want that, or comment on one if it exists).

Note that when opening a file in blocking mode, such as a git commit, the terminal will be inaccessible. You can get the filetype from the bufnr or filetype arguments of the `post_open` callback to only close the terminal for blocking files, and the `block_end` callback to reopen it afterwards.

Here's my setup for toggleterm, including an autocmd to automatically close a git commit buffer on write:

```lua
{
    'willothy/flatten.nvim',
    opts = {
        window = {
            open = "alternate"
        },
        callbacks = {
            post_open = function(bufnr, winnr, ft, is_blocking)
                if is_blocking then
                    -- Hide the terminal while it's blocking
                    require("toggleterm").toggle(0)
                else
                    -- If it's a normal file, just switch to its window
                    vim.api.nvim_set_current_win(winnr)
                end

                -- If the file is a git commit, create one-shot autocmd to delete its buffer on write
                -- If you just want the toggleable terminal integration, ignore this bit
                if ft == "gitcommit" then
                    vim.api.nvim_create_autocmd(
                        "BufWritePost",
                        {
                            buffer = bufnr,
                            once = true,
                            callback = function()
                                -- This is a bit of a hack, but if you run bufdelete immediately
                                -- the shell can occasionally freeze
                                vim.defer_fn(
                                    function()
                                        vim.api.nvim_buf_delete(bufnr, {})
                                    end,
                                    50
                                )
                            end
                        }
                    )
                end
            end,
            block_end = function()
                -- After blocking ends (for a git commit, etc), reopen the terminal
                require("toggleterm").toggle(0)
            end
        }
    }
}

```

## About

The name is inspired by the flatten function in Rust (and maybe other languages?), which flattens nested types (`Option<Option<T>>` -> `Option<T>`, etc).

The plugin itself is inspired by [`nvim-unception`](https://github.com/samjwill/nvim-unception), which accomplishes the same goal but functions a bit differently and doesn't allow as much configuration.

[^1]: Lazy loading this plugin is not recommended - flatten should always be loaded as early as possible. Starting the host is essentially overhead-free other than the setup() function as it leverages the RPC server started on init by Neovim, and loading plugins before this in a guest session will only result in poor performance.
