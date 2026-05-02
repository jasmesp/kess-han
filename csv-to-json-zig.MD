# Building a CSV to JSON Tool in Zig

## Required Versions

Use these exact tool versions for this assignment:

| Tool | Required version |
| --- | --- |
| Zig compiler | `0.16.0` stable release |
| ZLS language server | `0.16.0` stable release |
| CSV package | `zcsv 0.8.0`, pinned to commit `06b18644ef293b9b2def9885a9b460026fc420c4` |

Do not use Zig `master`, a nightly build, or a `0.16.0-dev...` build for this assignment. Do not use a ZLS `0.16.0-dev...` build either. The compiler and language server should both report the stable `0.16.0` version.

Check your tools before starting:

```bash
zig version
zls --version
```

Expected output:

```text
0.16.0
0.16.0
```

If either command prints a different version, fix the tool version before continuing. Zig examples are sensitive to compiler version, and ZLS should match the Zig release used by the project.

Recommended version manager: ZVM (`tristanisham/zvm`). It can install Zig and ZLS together, which helps keep this assignment on the required `0.16.0` toolchain even if another project requires a different Zig version.

```bash
zvm install --zls 0.16.0
zvm use 0.16.0
```

Supported editor paths: use either the course Neovim setup or VS Code for this assignment. Those are the two editor paths this handout is written to support directly. Other editors are bring-your-own-editor setups: they may work, but they add version-management and language-server setup details that students are responsible for resolving.

## VS Code Setup

Students using VS Code should install the official Zig extension:

```bash
code --install-extension ziglang.vscode-zig
```

Use the extension named **Zig Language** with the identifier `ziglang.vscode-zig`. Do not install the older `AugusteRame.zls-vscode` extension; it is deprecated.

VS Code and ZVM do not conflict if VS Code uses the same `zig` and `zls` binaries that ZVM installed. The main problem to avoid is letting VS Code install or use a different Zig/ZLS pair than the required `0.16.0` versions for this assignment.

Recommended workspace settings for macOS or Linux:

```json
{
  "zig.path": "${env:HOME}/.zvm/bin/zig",
  "zig.zls.path": "${env:HOME}/.zvm/bin/zls",
  "zig.zls.enabled": "on",
  "zig.formattingProvider": "zls"
}
```

Recommended workspace settings for Windows:

```json
{
  "zig.path": "${env:USERPROFILE}\\.zvm\\bin\\zig.exe",
  "zig.zls.path": "${env:USERPROFILE}\\.zvm\\bin\\zls.exe",
  "zig.zls.enabled": "on",
  "zig.formattingProvider": "zls"
}
```

Put these settings in `.vscode/settings.json` for the assignment workspace. Workspace settings are preferred because they affect this project without changing a student's global VS Code setup.

After changing settings, restart VS Code or run **ZLS language server: Start / Restart Language Server** from the Command Palette. Then open a terminal inside VS Code and confirm:

```bash
zig version
zls --version
```

Both commands should print `0.16.0`. If VS Code shows a different version than the system terminal, VS Code is using a different `PATH`; keep the explicit `zig.path` and `zig.zls.path` settings above.

This tutorial guides you through building a small command-line tool that reads a CSV file and writes a JSON file. It intentionally does not give you the complete final program. The goal is to help you understand the structure, identify important parsing cases, and implement the missing pieces from a clear outline.

Zig's standard library includes JSON support through `std.json`, but it does not currently include a CSV parser. For CSV, this tutorial uses the community library `zcsv` from `matthewtolman/zig_csv`.

## Why Use A CSV Library

It may seem reasonable to split each line on commas:

```zig
// This only handles rows without quoted commas or quoted newlines.
var fields = std.mem.splitScalar(u8, line, ',');
```

That approach only handles a restricted subset of CSV files: rows where commas are always separators and newlines are always row endings. CSV files can also contain quoted fields, escaped quotes, commas inside values, and newlines inside quoted fields.

For this study tool, use a maintained CSV library instead of writing a parser from scratch. That lets you focus on the data transformation:

```text
CSV rows with headers -> row maps -> JSON array of objects
```

## What The Tool Should Do

Your tool should accept an input CSV path and an output JSON path:

```bash
zig build run -- input.csv output.json
```

At a high level, it should:

1. Open the CSV file.
2. Parse the first row as headers.
3. Convert each later row into a JSON object.
4. Write all objects inside a JSON array.
5. Report useful errors when input is missing or malformed.

The JSON output will usually look like this:

```json
[
  {
    "name": "Ada",
    "language": "Zig"
  },
  {
    "name": "Grace",
    "language": "COBOL"
  }
]
```

## Project Setup

Start with a normal Zig executable project:

```bash
zig init -m
```

Add the pinned `zcsv` dependency:

```bash
zig fetch --save git+https://github.com/matthewtolman/zig_csv#06b18644ef293b9b2def9885a9b460026fc420c4
```

That command should add a dependency entry like this to `build.zig.zon`:

```zig
.dependencies = .{
    .zcsv = .{
        .url = "git+https://github.com/matthewtolman/zig_csv#06b18644ef293b9b2def9885a9b460026fc420c4",
        .hash = "zcsv-0.8.0-Wao6Fl2sAgCxu_ObEjjK9KTD1yp8e4KDUpNMKFJNU7D8",
    },
},
```

If `build.zig.zon` contains `#main` instead of the commit hash, replace it with the pinned URL above. If `minimum_zig_version` is present, it should be compatible with `0.16.0`; this assignment is not written for development builds.

Then add the dependency to `build.zig` so the executable can import and use the `zcsv` module:

```zig
const zcsv = b.dependency("zcsv", .{
    .target = target,
    .optimize = optimize,
});

exe.root_module.addImport("zcsv", zcsv.module("zcsv"));
```

These lines make the dependency available to your project as `@import("zcsv")`. For this assignment, use this project setup unless your instructor provides a different Zig version or package configuration.

What these lines do:

- `b.dependency("zcsv", ...)` asks Zig's build system for the package named `zcsv` from `build.zig.zon`.
- `.target = target` and `.optimize = optimize` build the dependency with the same target and optimization mode as your executable.
- `exe.root_module.addImport("zcsv", ...)` makes the package available in your code with `const zcsv = @import("zcsv");`.

## The Core Data Shape

Conceptually, each CSV row becomes an object keyed by the header row:

```json
{
  "name": "Ada",
  "language": "Zig"
}
```

In Zig, you can approach this in two useful ways:

- Stream JSON directly as each CSV row is parsed.
- Build an in-memory list of row objects, then stringify the whole thing.

For a first version of the converter, streaming JSON is often simpler because it avoids designing a custom dynamic object type before you need one.

## Suggested Program Structure

Keep the program organized by splitting the responsibilities:

```zig
fn convertCsvToJson(allocator: std.mem.Allocator, input_path: []const u8, output_path: []const u8) !void {
    // Open files.
    // Create a zcsv parser.
    // Create a std.json writer/stringifier.
    // Loop over rows and write JSON objects.
}

pub fn main() !void {
    // Allocate args, validate count, then call convertCsvToJson.
}
```

This structure separates command-line handling from conversion logic while keeping the project limited in scope.

## Reading CSV With zcsv

For files with a header row, use `zcsv`'s map parser. The map parser reads the first row as column names and then returns each later row as a map from column name to field value.

The basic parser setup looks like this:

```zig
const zcsv = @import("zcsv");

var parser = try zcsv.allocs.map.init(allocator, input_reader, .{});
defer parser.deinit();

while (parser.next()) |row| {
    defer row.deinit();

    const fields = row.data();
    const name_field = fields.get("name") orelse return error.MissingName;
    const name = zcsv.decode.fieldToDecodedStr(name_field) orelse "";
    _ = name;
}

if (parser.err) |err| {
    return err;
}
```

Important `zcsv` map parser behavior:

- `zcsv.allocs.map.init(allocator, input_reader, .{})` creates a parser and reads the header row immediately.
- `parser.next()` returns the next data row, or `null` when there are no more rows.
- `row.data()` returns a map from header names to CSV field values.
- `fields.get("name")` retrieves the field for the `name` column, or returns `null` if that column is missing.
- `zcsv.decode.fieldToDecodedStr(field)` converts a CSV field into a string slice with CSV quoting already decoded.
- `row.deinit()` must be called for each row because the row owns memory.
- `parser.deinit()` must be called when the parser is no longer needed.
- `parser.err` should be checked after the loop because parsing errors can be discovered after some rows have already been returned.

If you want to preserve every column, iterate over the row map instead of naming columns one by one:

```zig
var iter = fields.iterator();
while (iter.next()) |entry| {
    const header = entry.key_ptr.*;
    const field = entry.value_ptr.*;
    const value = zcsv.decode.fieldToDecodedStr(field) orelse "";
    _ = header;
    _ = value;
}
```

Maps do not guarantee that fields will be visited in the original CSV column order. For JSON objects, field order is usually not important.

## Writing JSON With std.json

Zig's `std.json` can write JSON values and also supports lower-level streaming. For this project, use a JSON writer/stringifier and call methods in this order:

```zig
try json_writer.beginArray();

while (parser.next()) |row| {
    try json_writer.beginObject();
    // For each CSV field:
    //   write the header as the JSON object field name
    //   write the CSV value as a JSON string
    try json_writer.endObject();
}

try json_writer.endArray();
```

The method calls above are the important part of the design:

- `beginArray()` writes the opening `[` for the top-level JSON array.
- `beginObject()` writes `{` for one CSV row.
- `objectField(header)` writes a JSON object key.
- `write(value)` writes the field value and escapes strings correctly.
- `endObject()` closes the current row object.
- `endArray()` closes the top-level array.

Do not build JSON by concatenating strings. Let `std.json` handle quotes, backslashes, newlines, and other escaping rules.

## Command-Line Arguments

Use Zig's standard process helpers to read arguments:

```zig
var gpa: std.heap.GeneralPurposeAllocator(.{}) = .init;
defer _ = gpa.deinit();
const allocator = gpa.allocator();

const args = try std.process.argsAlloc(allocator);
defer std.process.argsFree(allocator, args);
```

Then check that the user passed exactly two paths after the program name. If not, print a short usage message and return an error.

## Footguns And Edge Cases

Watch for these before you call the tool finished:

- Zig version drift: examples for different Zig versions may not compile unchanged.
- Library version drift: pin a tag or commit once your project works instead of depending on the changing `main` branch.
- Empty CSV files: a header-based parser may fail during initialization.
- Duplicate headers: a map cannot represent two different columns with the same key.
- Blank headers: empty strings as JSON keys are legal but usually not useful.
- Missing columns: decide whether to reject the row or emit an empty string.
- Extra columns: decide whether to reject them or preserve them somehow.
- Data types: CSV fields are text. Do not silently turn `"0012"` into `12` unless that is really intended.
- Memory ownership: many Zig parser rows must be deinitialized after each loop iteration.
- Partial reads: streaming parsers can emit valid rows before discovering a malformed row later.
- JSON validity: let `std.json` escape strings; do not concatenate JSON by hand.

## Library Choice Notes

This tutorial uses `zcsv` because it supports:

- Header-based map parsing.
- Quoted-field handling.
- Embedded newline support.
- Delimiter, quote, and newline customization.
- Allocating and zero-allocation parser options.

There are other Zig CSV libraries, including `DISTREAT/zig-csv`, which has more recent commits and Zig 0.15.2 metadata. For this assignment, keep using `zcsv` unless your instructor gives you a different parser. The important requirement is that the parser handles quoted commas, escaped quotes, and embedded newlines.

## Good Student Challenges

After the basic version works, try adding one improvement at a time:

- Add a `--pretty` flag for indented JSON.
- Reject duplicate or blank headers with a clear message.
- Add a `--limit 10` option for quick testing.
- Pin the CSV dependency to a tag instead of `main`.
- Add a test CSV with quoted commas and embedded quotes.
- Convert selected columns from strings to numbers only when explicitly requested.

Keep the first version limited in scope. Zig already asks you to think carefully about allocation, ownership, and errors. Let the CSV library handle CSV syntax while you practice connecting parser output to JSON output.
