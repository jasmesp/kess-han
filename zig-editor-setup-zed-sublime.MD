# Zig Editor Setup For Zed And Sublime Text

This handout is for students using Zed or Sublime Text for the Zig CSV-to-JSON assignment.

The assignment requires these versions:

| Tool | Required version |
| --- | --- |
| Zig compiler | `0.16.0` stable release |
| ZLS language server | `0.16.0` stable release |

Use ZVM (`tristanisham/zvm`) to install both tools:

```bash
zvm install --zls 0.16.0
zvm use 0.16.0
```

Then confirm:

```bash
zig version
zls --version
```

Both commands must print:

```text
0.16.0
0.16.0
```

## Suitability And Version Management

Zed and Sublime Text are bring-your-own-editor setups for this assignment. They are acceptable only if they are configured to use the ZVM-installed `zig` and `zls` binaries.

Supported editor paths are VS Code and the course Neovim setup. Zed and Sublime are allowed, but students using them are responsible for confirming that the editor is using Zig `0.16.0` and ZLS `0.16.0`, and for resolving editor-specific setup problems.

Do not let either editor install or select a different Zig or ZLS version for this assignment. If the editor cannot be configured to use the required `0.16.0` tools, treat that editor setup as unsuitable for this assignment and use VS Code or the course Neovim setup instead.

## Paths To Use

If ZVM is installed in its normal location, the binaries are:

macOS or Linux:

```text
/Users/YOUR_USER/.zvm/bin/zig
/Users/YOUR_USER/.zvm/bin/zls
```

Windows:

```text
C:\Users\YOUR_USER\.zvm\bin\zig.exe
C:\Users\YOUR_USER\.zvm\bin\zls.exe
```

On Linux, use `/home/YOUR_USER/.zvm/bin/zig` and `/home/YOUR_USER/.zvm/bin/zls` instead. Replace `YOUR_USER` with your actual username. If those paths do not exist, run the ZVM install commands again or ask for help before changing editor settings.

## Zed Setup

Zed has a Zig extension that provides language support for Zig. Zed's Zig support uses Tree-sitter for syntax parsing/highlighting and ZLS for language-server features such as completion, hover, diagnostics, and go-to-definition.

Install the Zig extension from Zed's extensions panel. In Zed, open the Command Palette and search for:

```text
zed: extensions
```

Install the extension named `Zig`.

Zed may automatically install a tagged ZLS release. For this assignment, do not rely on that automatic choice. Configure Zed to use the ZVM-installed `zls` and `zig` binaries instead.

Create or edit `.zed/settings.json` in the project folder.

macOS or Linux:

```json
{
  "languages": {
    "Zig": {
      "language_servers": ["zls"]
    }
  },
  "lsp": {
    "zls": {
      "binary": {
        "path": "/Users/YOUR_USER/.zvm/bin/zls"
      },
      "settings": {
        "zls": {
          "zig_exe_path": "/Users/YOUR_USER/.zvm/bin/zig"
        }
      }
    }
  }
}
```

Windows:

```json
{
  "languages": {
    "Zig": {
      "language_servers": ["zls"]
    }
  },
  "lsp": {
    "zls": {
      "binary": {
        "path": "C:\\Users\\YOUR_USER\\.zvm\\bin\\zls.exe"
      },
      "settings": {
        "zls": {
          "zig_exe_path": "C:\\Users\\YOUR_USER\\.zvm\\bin\\zig.exe"
        }
      }
    }
  }
}
```

On Linux, replace `/Users/YOUR_USER` with `/home/YOUR_USER`.

After changing settings, restart Zed. Open the integrated terminal and run:

```bash
zig version
zls --version
```

If the terminal prints `0.16.0` for both tools but Zed diagnostics still look wrong, open Zed's language server logs and confirm that it started the `zls` binary from the ZVM path.

## Sublime Text Setup

Sublime Text can work for this assignment, but it requires two packages:

- `LSP`, which adds Language Server Protocol support to Sublime.
- `sublime-zig-language`, which adds Zig syntax support and Zig-specific editor behavior.

Install both packages through Package Control.

Then configure Sublime's LSP package to start the ZVM-installed ZLS binary and pass it the ZVM-installed Zig compiler path.

Open:

```text
Preferences -> Package Settings -> LSP -> Settings
```

Add a Zig client configuration.

macOS or Linux:

```json
{
  "clients": {
    "zls": {
      "enabled": true,
      "command": ["/Users/YOUR_USER/.zvm/bin/zls"],
      "selector": "source.zig",
      "settings": {
        "zls": {
          "zig_exe_path": "/Users/YOUR_USER/.zvm/bin/zig"
        }
      }
    }
  },
  "lsp_format_on_save": true
}
```

Windows:

```json
{
  "clients": {
    "zls": {
      "enabled": true,
      "command": ["C:\\Users\\YOUR_USER\\.zvm\\bin\\zls.exe"],
      "selector": "source.zig",
      "settings": {
        "zls": {
          "zig_exe_path": "C:\\Users\\YOUR_USER\\.zvm\\bin\\zig.exe"
        }
      }
    }
  },
  "lsp_format_on_save": true
}
```

On Linux, replace `/Users/YOUR_USER` with `/home/YOUR_USER`.

Then configure the Zig package so that Sublime does not also try to run its own formatter separately from ZLS.

Open:

```text
Preferences -> Package Settings -> Zig -> Settings
```

Use:

```json
{
  "zig.fmt.on_save": false,
  "zig.quiet": true
}
```

After changing settings, restart Sublime Text. Open a `.zig` file and confirm that the LSP package starts `zls`.

## What ZLS Provides

ZLS is the Zig language server. When the editor is configured correctly, it can provide:

- Completion suggestions.
- Hover information.
- Go-to-definition.
- Diagnostics while editing.
- Formatting through `zig fmt`.
- Some code actions, depending on editor support.

Syntax highlighting is separate from ZLS:

- Zed uses the Zig extension's language support, including Tree-sitter grammar support.
- Sublime uses the `sublime-zig-language` package for Zig syntax support.

## Troubleshooting Checklist

Use this checklist before asking for help:

- `zig version` prints `0.16.0`.
- `zls --version` prints `0.16.0`.
- The editor settings point to the ZVM-installed `zig` and `zls` paths.
- The project has been reopened after changing editor settings.
- The editor is not using an automatically installed Zig or ZLS version.
- The Zig file has a `.zig` extension.
- The project folder is opened at the assignment root, not just an individual file.

If Zed or Sublime still cannot use the required tool versions after these checks, use VS Code or the course Neovim setup for this assignment.

## Research Notes

Zed is acceptable because its Zig extension provides Zig language support and the Zed language documentation lists `tree-sitter-zig` and `zls` for Zig. The ZLS editor notes also document Zed configuration and warn that Zed may automatically install a tagged ZLS release.

Sublime Text is acceptable because the ZLS editor notes document a setup using Sublime's `LSP` package plus `sublime-zig-language`. This setup is more manual than VS Code or Neovim, but it is not unsuitable as long as it uses the pinned ZVM-managed tools.
