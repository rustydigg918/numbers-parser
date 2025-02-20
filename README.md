# numbers-parser

[![build:](https://travis-ci.com/masaccio/numbers-parser.svg?branch=master)](https://app.travis-ci.com/github/masaccio/numbers-parser)

`numbers-parser` is a Python module for parsing [Apple Numbers](https://www.apple.com/numbers/)
`.numbers` files. It supports Numbers files generated by Numbers version 10.3, and all 11.x up to 11.2
(current as of November 2021).

It supports and is tested against Python versions from 3.6 onwards. It is not compatible with earlier versions of Python.

Currently supported features of Numbers files are:

* Multiple sheets per document
* Multiple tables per sheet
* Text, numeric, date, currency, duration, percentage cell types

Formulas have very limited support and rely wholly on Numbers saving values in cells as part of the saved document,
which is not always guaranteed. When a formula value is not present, the value `*FORMULA*` is returned. Any formula
that results in a Numbers error returns a value `*ERROR*`.

## Installation

``` bash
python3 -m pip install numbers-parser
```

## Usage

Reading documents:

``` python
from numbers_parser import Document
doc = Document("my-spreasdsheet.numbers")
sheets = doc.sheets()
tables = sheets[0].tables()
rows = tables[0].rows()
```

### Referring to sheets and tables

Both sheets and names can be accessed from lists of these objects using an integer index (`list` syntax) and using the name
of the sheet/table (`dict` syntax):

``` python
# list access method
sheet_1 = doc.sheets()[0]
print("Opened sheet", sheet_1.name)

# dict access method
table_1 = sheets["Table 1"]
print("Opened table", table_1.name)
```

### Accessing data

`Table` objects have a `rows` method which contains a nested list with an entry for each row of the table. Each row is
itself a list of the column values. Empty cells in Numbers are returned as `None` values.

``` python
data = sheets["Table 1"].rows()
print("Cell A1 contains", data[0][0])
print("Cell C2 contains", data[2][1])
```

### Cell references

In addition to extracting all data at once, individual cells can be referred to as methods

``` python
doc = Document("my-spreasdsheet.numbers")
sheets = doc.sheets()
tables = sheets["Sheet 1"].tables()
table = tables["Table 1"]

# row, column syntax
print("Cell A1 contains", table.cell(0, 0))
# Excel/Numbers-style cell references
print("Cell C2 contains", table.cell("C2"))
```

### Merged cells

When extracting data using ```data()``` merged cells are ignored since only text values
are returned. The ```cell()``` method of ```Table``` objects returns a ```Cell``` type
object which is typed by the type of cell in the Numbers table. ```MergeCell``` objects
indicates cells removed in a merge.

``` python
doc = Document("my-spreasdsheet.numbers")
sheets = doc.sheets()
tables = sheets["Sheet 1"].tables()
table = tables["Table 1"]

cell = table.cell("A1")
print(cell.merge_range)
print(f"Cell A1 merge size is {cell.size[0]},{cell.size[1]})
```

### Row and column iterators

Tables have iterators for row-wise and column-wise iteration with each iterator
returning a list of the cells in that row or column

``` python
for row in table.iter_rows(min_row=2, max_row=7, values_only=True):
    sum += row
for col in table.iter_cole(min_row=2, max_row=7):
    sum += col.value
```

### Pandas

Since the return value of `data()` is a list of lists, you should be able to pass it straight to pandas like this

``` python
import pandas as pd

doc = Document("simple.numbers")
sheets = doc.sheets()
tables = sheets[0].tables()
data = tables[0].rows(values_only=True)
df = pd.DataFrame(data, columns=["A", "B", "C"])
```

### Bullets and lists

Cells that contain bulleted or numbered lists can be identified by the `is_bulleted` property. Data from such cells is returned using the `value` property as with other cells, but can additionally extracted using the `bullets` property. `bullets` returns a list of the paragraphs in the cell without the bullet or numbering character. It is not possible to distingush between bulleted and numbered lists; each is handled identically. Each bullet in th elist retains its newline character.

``` python
doc = Document("bullets.numbers")
sheets = doc.sheets()
tables = sheets[0].tables()
table = tables[0]
if not table.cell(0, 1).is_bulleted:
    print(table.cell(0, 1).value)
else:
    bullets = ["* " + s for s in table.cell(0, 1).bullets]
    print("".join(bullets))
```

## Numbers File Formats

Numbers uses a proprietary, compressed binary format to store its tables.
This format is comprised of a zip file containing images, as well as
[Snappy](https://github.com/google/snappy)-compressed
[Protobuf](https://github.com/protocolbuffers/protobuf) `.iwa` files containing
metadata, text, and all other definitions used in the spreadsheet.

### Protobuf updates

As `numbers-parser` includes private Protobuf definitions extracted from a copy of Numbers,
new versions of Numbers will inevitably create `.numbers` files that cannot be read by `numbers-parser`.
As new versions of Numbers are released, the following steps must be undertaken:

* Run [proto-dump](https://github.com/masaccio/proto-dump) on the new copy of Numbers to dump
  new Proto files.
  * proto-dump assumes version 2.5.0 of Google Protobuf  which may need changes to build on more modern OSes.  The version linked here is maintained by the author and tested on recent macOS for both arm64 and x86_64 architectures.
  * Any `.` characters in the Protobuf definitions must be changed to `_` characters manually, or via the `rename_proto_files.py` script in the `protos` directory of this repo.
* Connect to a running copy of `Numbers` with `lldb` (or any other debugger) and manually copy
  and reformat the results of `po [TSPRegistry sharedRegistry]` into `mapping.py`.
  * Versions of macOS >= 10.11 may protect Numbers from being attached to by a debugger -
    to attach, [temporarily disable System IntegrityProtection](https://developer.apple.com/documentation/security/disabling_and_enabling_system_integrity_protection)
    to get this data.
  * The `generate_mapping.py` script in `protos` should help turn the output from this step into a
    recreation of `mapping.py`

Running `make bootstrap` will perform all of these steps and generate the Python protos files as
well as `mapping.py`. The makefile assumes that [proto-dump](https://github.com/masaccio/proto-dump)
is in a repo parallel to this one, but the make variable `PROTO_DUMP` can be overridden to pass
the path to a working version of `proto-dump`.

## Credits

`numbers-parser` was built by [Jon Connell](http://github.com/masaccio) but derived enormously
from [prior work](https://github.com/psobot/keynote-parser) by [Peter Sobot](https://petersobot.com).
Both modules are derived from [previous work](https://github.com/obriensp/iWorkFileFormat/blob/master/Docs/index.md)
by [Sean Patrick O'Brien](http://www.obriensp.com).

Decoding the data structures inside Numbers files was helped greatly by
[previous work](https://github.com/slott56/Stingray-Reader) by [Steven Lott](https://github.com/slott56).

Formula tests were adapted from JavaScript tests used in
[fast-formula-parser](https://github.com/LesterLyu/fast-formula-parser).

## License

All code in this repository is licensed under the [MIT License](https://github.com/masaccio/numbers-parser/blob/master/LICENSE.rst)
