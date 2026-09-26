# Testing generated files (xlsx / csv exports) in Pest

An export test that only asserts `assertOk()` and `assertDownload()` proves the
route returns *a* file. It does not prove the file has the right rows, headings or
ordering — which is the whole point of the feature. Read the artifact back.

## Download, then parse

```php
$response = $this->get(route('units.export'));
$response->assertOk();

$path = $response->baseResponse->getFile()->getPathname();
// BinaryFileResponse points at a temp file the framework cleans up per request.
copy($path, $tmp = tempnam(sys_get_temp_dir(), 'export-').'.xlsx');

$spreadsheet = \PhpOffice\PhpSpreadsheet\IOFactory::load($tmp);
$rows = $spreadsheet->getActiveSheet()->toArray(null, true, true, true);
array_shift($rows);                       // drop the headings row
$spreadsheet->disconnectWorksheets();
```

## The `toArray` gotcha

`Worksheet::toArray($cellValue, $formatData, $returnCellRef, $calculateFormulas)`.
The third argument is the one that bites:

| 3rd arg | Row keys | Use for |
|---|---|---|
| `false` | `0, 1, 2…` | positional checks |
| `true` | `'A', 'B', 'C'…` | reading a column by heading |

Reading a column by index (`$row[$i + 1]`) when you asked for cell refs returns
`null` for every row — the test fails while the real code is fine. Pick one style:

```php
// by heading
$letter = Coordinate::stringFromColumnIndex($i + 1);
$column = array_map(fn ($row) => $row[$letter] ?? null, $rows);
```

Wrap the read in helpers (`rowsFromRoute()` / `column($rows, $heading)`) so a
single mistake shows up uniformly across every test that uses it — a shared
helper bug then reads as one obvious fault instead of scattered feature bugs.

## Numbers come back as strings

A cell written as an int reads back as `'0'`, `'1'`, `'2'`. Cast before asserting
when the assertion is about magnitude:

```php
$this->assertSame([0, 1, 2], array_map('intval', $this->column($rows, 'سطح')));
```

## Prefer the real file over `Excel::fake()`

`Excel::fake()` records the `Export` object without running the writer, so
`WithEvents` listeners, RTL settings and heading/mapping wiring are never
exercised. For a feature whose deliverable *is* the file, download it.

`Excel::fake()` is still right when you only need to assert *that* an export was
requested (dispatch, job, filename) without paying for a real write.

## Keep the diagnostic small

A failed regex or `assertSee` against a full HTML/xlsx dump prints the whole
document and makes the test look like a hang. Assert on a boolean with a one-line
message instead:

```php
$this->assertTrue(
    (bool) preg_match('/<a[^>]+href="'.preg_quote(route('x.export'), '/').'"[^>]*>/', $html),
    'Export control must be an <a href> pointing at the export route.'
);
```

## What the tests should cover

Route guard (guest redirect, permission 403), access scoping (only the caller's
subtree; empty scope yields headers only, not an error), per-row content,
parent-before-child ordering, cycle safety, and that the UI control is an anchor.
Column content and ordering need the parse above — `assertDownload()` covers
status and headers only.
