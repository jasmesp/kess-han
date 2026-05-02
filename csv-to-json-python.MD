# Building a CSV to JSON Tool in Python

This tutorial guides you through building a small command-line tool that reads a CSV file and writes a JSON file. It intentionally does not give you the complete final program. The goal is to help you understand the shape of the solution, spot common traps, and practice implementing the missing pieces from a clear outline.

You will use Python's standard library for the actual conversion and `uv` for project setup and running commands.

## What The Tool Should Do

Your tool should accept an input CSV path and an output JSON path:

```bash
uv run csv_to_json.py input.csv output.json
```

At a high level, it should:

1. Open the CSV file.
2. Read the first row as column headers.
3. Convert each following row into a dictionary.
4. Store all row dictionaries in a list.
5. Write that list as JSON.

The JSON output will usually look like this:

```json
[
  {
    "name": "Ada",
    "language": "Python"
  },
  {
    "name": "Linus",
    "language": "C"
  }
]
```

## Project Setup With uv

Create a small Python project or script workspace with `uv`:

```bash
uv init csv-json-python
cd csv-json-python
```

You do not need third-party packages for this project. Python already includes the `csv`, `json`, `argparse`, and `pathlib` modules.

If you are working from an existing folder instead of creating a new one, you can still run your script with:

```bash
uv run python csv_to_json.py input.csv output.json
```

For a small study tool, it is fine to start with one file named `csv_to_json.py`.

## The Core Data Shape

In Python, a CSV row maps naturally to a dictionary:

```python
row = {
    "name": "Ada",
    "language": "Python",
}
```

A whole CSV file then becomes a list of dictionaries:

```python
records = [
    {"name": "Ada", "language": "Python"},
    {"name": "Linus", "language": "C"},
]
```

Python's `csv.DictReader` can create this shape for you when the CSV has a header row.

## Suggested Program Structure

Try organizing your program around small functions:

```python
from pathlib import Path


def read_csv(path: Path) -> list[dict[str, str]]:
    # Open the file and use csv.DictReader here.
    ...


def write_json(path: Path, records: list[dict[str, str]]) -> None:
    # Open the file and use json.dump here.
    ...


def main() -> None:
    # Parse command-line arguments, then call the two functions above.
    ...
```

This structure gives you clear places to test and debug. If your program fails, you can check whether the problem is in reading, transforming, or writing.

## Reading CSV

Use `csv.DictReader` to read the CSV. `DictReader` treats the first row as column names. Each later row becomes a dictionary where the keys are those column names and the values are the row's cell values.

For example, this CSV:

```csv
name,language
Ada,Python
Linus,C
```

becomes dictionaries shaped like this:

```python
{"name": "Ada", "language": "Python"}
{"name": "Linus", "language": "C"}
```

A partial example:

```python
import csv

with input_path.open("r", encoding="utf-8", newline="") as csv_file:
    reader = csv.DictReader(csv_file)
    records = list(reader)
```

The `newline=""` argument is recommended when using Python's `csv` module. It lets the CSV module handle line endings consistently across platforms.

Useful `DictReader` behavior to know:

- `reader.fieldnames` contains the column names after the reader has started reading.
- Values are read as strings, not as integers, floats, or booleans.
- If a row has fewer values than the header row, the missing columns get `None`.
- If a row has more values than the header row, the extra values are stored under a special key unless you configure `restkey`.

## Writing JSON

Use `json.dump` to write JSON to a file:

```python
import json

with output_path.open("w", encoding="utf-8") as json_file:
    json.dump(records, json_file, indent=2)
```

The `indent=2` option makes the output easier for humans to read. For very large files, pretty JSON creates a larger output file, so compact JSON may be better.

Useful `json.dump` behavior to know:

- It writes a Python value directly to an open file.
- Lists become JSON arrays.
- Dictionaries become JSON objects.
- Strings, numbers, booleans, and `None` become their JSON equivalents.
- `indent=2` adds line breaks and spaces for readable output.

## Command-Line Arguments

For a real tool, avoid hard-coding file names. The standard `argparse` module is a good fit:

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("input_csv")
parser.add_argument("output_json")
args = parser.parse_args()
```

Then convert those strings into `Path` objects before passing them to your functions.

In this example, `add_argument("input_csv")` creates a required positional argument. The first path after the script name becomes `args.input_csv`, and the second becomes `args.output_json`.

## Footguns And Edge Cases

Watch for these before you call the tool finished:

- Empty CSV files: `DictReader` will not have useful headers if the file is empty.
- Missing headers: a CSV without a first-row header will produce confusing keys.
- Duplicate headers: dictionaries cannot keep two separate values for the same key.
- Extra columns: `DictReader` may place extra values under a special key unless you handle them.
- Missing columns: some row values may become `None`.
- Data types: CSV values arrive as strings, even when they look like numbers or booleans.
- Encoding: `utf-8` is a good default, but real-world CSV files may use other encodings.
- Large files: `list(reader)` loads the entire CSV into memory. That is fine for small files, but not ideal for huge datasets.

## Good Student Challenges

After the basic version works, try adding one improvement at a time:

- Add a `--pretty` flag that turns indentation on or off.
- Validate that the input file exists before reading it.
- Print a helpful error if the CSV has no headers.
- Add a `--limit 10` option for testing with only the first few rows.
- Convert selected columns from strings to numbers.

Keep each improvement small. The best debugging habit is changing one thing, testing it, and then moving on.
