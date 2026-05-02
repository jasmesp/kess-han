# Building a CSV to JSON Tool in Zig

This tutorial guides you through building a small command-line tool that reads a CSV file and writes a JSON file. It intentionally does not give you the complete final program. The goal is to help you understand the structure, avoid the painful edge cases, and practice checking library docs as you go.

Zig's standard library includes JSON support through `std.json`, but it does not currently include a CSV parser. For CSV, this tutorial uses the community library `zcsv` from `matthewtolman/zig_csv`.

Important version note: Zig changes quickly. Before copying any setup command, check the library README and choose a Zig version the library supports.

## Why Use A CSV Library

It is tempting to split each line on commas:

```zig
// This is not enough for real CSV.
var fields = std.mem.splitScalar(u8, line, ',');
```

That works only for the friendliest possible CSV files. Real CSV can contain quoted fields, escaped quotes, commas inside values, and sometimes newlines inside a quoted field.

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
zig init
```

Add `zcsv` as a dependency. The library README currently shows this package-manager shape:

```bash
zig fetch --save git+https://github.com/matthewtolman/zig_csv#main
```

Then wire the module into `build.zig`:

```zig
const zcsv = b.dependency("zcsv", .{
    .target = target,
    .optimize = optimize,
});

exe.root_module.addImport("zcsv", zcsv.module("zcsv"));
```

Do not treat this as permanent magic. If Zig or `zcsv` changes, the README is the source of truth.

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

For a beginner-friendly converter, streaming JSON is often better because it avoids designing a custom dynamic object type right away.

## Suggested Program Structure

Keep the program small, but split the responsibilities:

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

This is enough structure to keep your code from becoming one giant `main`, but still small enough for a study project.

## Reading CSV With zcsv

For files with a header row, start by looking at `zcsv`'s map parser. Its README describes the map parser as the parser for CSV files where the first row is a list of column names.

The rough shape looks like this:

```zig
const zcsv = @import("zcsv");

var parser = try zcsv.allocs.map.init(allocator, input_reader, .{});
defer parser.deinit();

while (parser.next()) |row| {
    defer row.deinit();

    const fields = row.data();
    // Pull values out by header name here.
}
```

This is intentionally incomplete. Your job is to inspect the row API and decide how to iterate over the key/value pairs or extract the columns your assignment cares about.

## Writing JSON With std.json

Zig's `std.json` can write JSON values and also supports lower-level streaming. For this project, a streaming shape is a good mental model:

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

Depending on your Zig version, the exact API may be `std.json.stringify`, `std.json.Stringify`, or a related writer helper. Look up `std.json` for the Zig version you are using instead of guessing from an old snippet.

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

- Zig version drift: examples for Zig 0.14, 0.15, and 0.16 may not compile unchanged.
- Library version drift: pin a tag or commit once your project works instead of depending on moving `main` forever.
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

As of this tutorial, `zcsv` is a reasonable choice because it documents:

- Header-based map parsing.
- Quoted-field handling.
- Embedded newline support.
- Delimiter, quote, and newline customization.
- Allocating and zero-allocation parser options.

There are other Zig CSV libraries, including `DISTREAT/zig-csv`, which has more recent commits and Zig 0.15.2 metadata. If you compare libraries, do not look only at activity. Check whether the parser handles quoted commas, escaped quotes, and embedded newlines. A maintained library that only splits rows and delimiters may still be a poor fit for real CSV data.

## Good Student Challenges

After the basic version works, try adding one improvement at a time:

- Add a `--pretty` flag for indented JSON.
- Reject duplicate or blank headers with a clear message.
- Add a `--limit 10` option for quick testing.
- Pin the CSV dependency to a tag instead of `main`.
- Add a test CSV with quoted commas and embedded quotes.
- Convert selected columns from strings to numbers only when explicitly requested.

Keep the first version boring. Zig already asks you to think carefully about allocation, ownership, and errors. Let the CSV library handle CSV syntax while you practice connecting parser output to JSON output.

## References To Check

- `zcsv` repository and README: https://github.com/matthewtolman/zig_csv
- `DISTREAT/zig-csv` repository for comparison: https://github.com/DISTREAT/zig-csv
- Zig standard library docs: https://ziglang.org/documentation/master/std/
