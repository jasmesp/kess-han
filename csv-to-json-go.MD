# Building a CSV to JSON Tool in Go

This tutorial guides you through building a small command-line tool that reads a CSV file and writes a JSON file. It does not include the entire finished program. Instead, it gives you the structure, important data shapes, and the common traps to watch for while you build.

For Go, use the Go toolchain for package management and running the program. `uv` is excellent for Python projects, but it is not the package manager for Go.

## What The Tool Should Do

Your tool should accept an input CSV path and an output JSON path:

```bash
go run . input.csv output.json
```

At a high level, it should:

1. Open the CSV file.
2. Read the first row as column headers.
3. Convert each later row into a map.
4. Store all row maps in a slice.
5. Write that slice as JSON.

The JSON output will usually look like this:

```json
[
  {
    "name": "Ada",
    "language": "Python"
  },
  {
    "name": "Ken",
    "language": "Go"
  }
]
```

## Project Setup

Create a small Go module:

```bash
mkdir csv-json-go
cd csv-json-go
go mod init example.com/csv-json-go
```

You do not need third-party packages for this project. Go already includes `encoding/csv`, `encoding/json`, `os`, and other useful standard-library packages.

For a small study tool, it is fine to start with one file named `main.go`.

## The Core Data Shape

In Go, one CSV row can become a `map[string]string`:

```go
record := map[string]string{
	"name":     "Ada",
	"language": "Python",
}
```

A whole CSV file can become a slice of those maps:

```go
records := []map[string]string{
	{"name": "Ada", "language": "Python"},
	{"name": "Ken", "language": "Go"},
}
```

This shape converts cleanly to JSON with `encoding/json`.

## Suggested Program Structure

Try organizing your program around small functions:

```go
func readCSV(path string) ([]map[string]string, error) {
	// Open the file, read headers, then build records.
}

func writeJSON(path string, records []map[string]string) error {
	// Create the output file and encode records as JSON.
}

func main() {
	// Read command-line args, call readCSV, then call writeJSON.
}
```

Returning errors instead of exiting inside helper functions makes the code easier to test and reason about.

## Reading CSV

Look up `encoding/csv.Reader` in the Go documentation. You will usually read the first row separately as headers:

```go
reader := csv.NewReader(file)

headers, err := reader.Read()
if err != nil {
	return nil, err
}
```

Then each later row can be matched with the headers:

```go
record := make(map[string]string)

for i, header := range headers {
	record[header] = row[i]
}
```

That loop is only safe if you have already checked that the row has enough columns. Real CSV files are often messier than classroom examples.

## Writing JSON

Use `encoding/json` to write the records:

```go
encoder := json.NewEncoder(file)
encoder.SetIndent("", "  ")
err := encoder.Encode(records)
```

`SetIndent` makes the output easier to inspect while learning. If the output might become huge, compact JSON may be better.

## Command-Line Arguments

For the simplest version, `os.Args` is enough:

```go
if len(os.Args) != 3 {
	fmt.Fprintln(os.Stderr, "usage: csv-json input.csv output.json")
	os.Exit(1)
}
```

The input path will be `os.Args[1]`, and the output path will be `os.Args[2]`.

For a larger tool, you might eventually use the standard `flag` package.

## Footguns And Edge Cases

Watch for these before you call the tool finished:

- Empty CSV files: reading headers will return an error, often `io.EOF`.
- Duplicate headers: a map cannot preserve two different values under the same key.
- Uneven rows: some rows may have fewer or more fields than the header row.
- Blank headers: an empty string can become a JSON key, which is legal but usually not useful.
- Data types: CSV values are strings, even when they look like numbers or booleans.
- File cleanup: remember to close files, usually with `defer file.Close()` after checking the open error.
- Error messages: wrap errors with context so users know which file failed.
- Large files: storing all records in memory is simple, but streaming may be better for huge CSV files.

## Handling Uneven Rows

Go's CSV reader has a `FieldsPerRecord` setting. The default behavior can be stricter than you expect. For a study project, choose your behavior deliberately:

- Strict mode: reject rows that do not match the header length.
- Lenient mode: fill missing values with empty strings and ignore extra values.
- Report mode: write a helpful error that includes the row number.

Strict mode is usually the easiest to reason about while learning.

## Good Student Challenges

After the basic version works, try adding one improvement at a time:

- Add a `-pretty` flag that controls JSON indentation.
- Validate duplicate or blank headers before reading the rest of the file.
- Include row numbers in error messages.
- Add tests for empty files and uneven rows.
- Convert selected columns from strings to numbers.

Keep each improvement small. Go rewards clear control flow, so prefer simple functions and explicit error handling over clever shortcuts.
