# Ledgerwright

A desk for the messy version of a business file.

Paste a lead export that nobody cleaned, or a pile of invoice scraps copied out of an inbox. Ledgerwright sends that text through one Grok pass and writes two things: a CSV you can open in a spreadsheet, and a one-page HTML report. Repairs are labeled. Computed figures are labeled. If the source did not actually say it, the cell stays empty.

The program is one file, `ledgerwright.py`. It uses the Python standard library and the xAI API. It does not install packages, and it does not store the API key.

## About me

I am Metta Thomas. On GitHub I am [M3TA](https://github.com/IAmM3ta). The short bio I keep on that profile is HomoSentien. The company line is 3VE Inc. The location I list is worldwide. The longer pointer is [linktr.ee/M3TAV](https://linktr.ee/M3TAV).

I build tools and write the technical record that goes with them. The public thread runs through living-field instruments and reaction-diffusion work ([morphogen](https://github.com/IAmM3ta/morphogen), [turing](https://github.com/IAmM3ta/turing), [MORPHOS](https://github.com/IAmM3ta/MORPHOS-APP)), EEG and cymatics ([MODE](https://github.com/IAmM3ta/MODE-Music-Ontological-Data-Extractor)), and the boring problem of not losing the thread in a long automation ([llm-context-preservation](https://github.com/IAmM3ta/llm-context-preservation)). A lot of the other work is hardware, mobility, and paper that is not in a public repo.

The habit under all of it is the same. Take a thing apart. Do not paper over a gap. If a number was inferred, say so. If a repair would be a guess, do not make it.

Ledgerwright is that habit pointed at a spreadsheet. Sales exports and invoice inboxes are where people quietly invent an email, a tax, or a merge so the file will import. This script refuses. A doubled dot in `jane@acme..example` is an unambiguous repair, so it is fixed and marked. `bobsmithgmail.com` could be more than one address, so it is left as written. A tax rate someone hedged with "I think" is not booked. A duplicate person stays on the sheet so you can see the merge before a CRM does it for you.

I would rather hand you a shorter, uglier, honest sheet than a clean one that made something up.

## How to use

### What you need

- Python 3.10 or newer. No pip install.
- An xAI API key in the environment. Get one from [console.x.ai](https://console.x.ai). The script reads `XAI_API_KEY` and never writes it to a file.

```bash
git clone https://github.com/IAmM3ta/ledgerwright.git
cd ledgerwright
export XAI_API_KEY="your key"
python ledgerwright.py check
```

`check` runs the local arithmetic and the "do not guess this email" tests. It does not call the API. You should see `self-check ok`.

### Clean a lead sheet

```bash
python ledgerwright.py leads samples/messy-leads.csv -o ./desk
```

A lead file can be a real CSV, a TSV, or a text export that is only mostly a table. The script keeps a header with each batch when the first row looks like a header.

You get:

- `desk/cleaned.csv` — one row per real person-row. Blank rows and a repeated header are dropped. Duplicates are not.
- `desk/report.html` — open it in a browser. It is a paper-colored ledger: headline, what to do next, then the table.

Lead columns:

| Column | Meaning |
| --- | --- |
| `source_row` | Where it sat in the intake, after blank rows and a repeated header are skipped. |
| `full_name`, `company`, `phone`, `source` | Cleaned casing and whitespace. Empty if the source had none. |
| `email` | Lowercased. Not rewritten unless the repair is unambiguous. |
| `email_status` | `valid`, `repaired`, `suspect`, or `missing`. |
| `captured_on` | `YYYY-MM-DD`, or empty. A "week of" phrase is not turned into a fake day. A two-digit year in the 00–69 range is read as 2000–2069. A missing year may be taken from the rest of the file, and that inference is listed in `issues`. |
| `interest` | `hot`, `warm`, `cold`, or `unknown`. Hot means a quote, approved budget, a security review, or a named quarter they will move. Warm means engaged without that. Cold means "later" or "not ready". The word "warm" in a note does not override an approved budget. |
| `next_action` | One imperative for this week. |
| `notes` | Only what the source supported. |
| `issues` | Repairs, truncations, inferred years. Empty if the row was clean. |
| `match_key` | How the report groups the same person. Same name plus the same email domain, or the same usable email. Rows are not deleted when they match. |

### Clean invoices, bills, and credits

```bash
python ledgerwright.py invoices samples/messy-invoices.txt -o ./desk
```

The input can be a forwarded inbox, a broken paste, OCR of a handwritten slip, or several documents split by a line of `----`.

`desk/cleaned.csv` is one row per line item, with the invoice header repeated, so a spreadsheet can pivot it. A document with no lines still gets one row. `desk/report.html` is one card per document: vendor, number, date, lines, subtotal, tax, total, paid, balance, and flags.

Money rules:

- A figure is computed only when the arithmetic is fully determined by numbers in the source. Quantity times a stated rate is allowed, and the report says the line amount was computed.
- A plainly stated tax rate may be computed, and it is flagged.
- A hedged rate ("I think", "they said", "about", a question mark) is not computed. Tax stays empty.
- "Tax included" does not mean tax is zero. Tax stays empty and the flag says it was inside the total.
- "No tax" or resale is tax `0`.
- A credit memo is stored as a negative total and a negative balance.
- Paid in full means status `paid`, amount paid equals total, balance `0`.
- A payment plus an unstated remaining balance is `partial`, and the balance cell stays empty rather than being invented.

### Useful flags

```bash
python ledgerwright.py leads messy.csv -o ./out --model grok-4.5
```

`--model` defaults to `grok-4.5`. You can also set `LEDGERWRIGHT_MODEL`. `-o` / `--out` defaults to `./ledgerwright-out`.

On a long lead sheet the script batches 15 data rows at a time and prints `pass 1 of N` to stderr. It stops at 90 rows so one command cannot quietly spend a large API bill. Split the file and run the pieces. Invoice scraps over 14,000 characters must be split on `----` lines, and a run will not send more than six chunks.

### What the run will not do

- It will not invent an email, a phone, a company, a date, a vendor, or a tax.
- It will not complete a truncated address unless the same row states the full address (a signature line counts, and the status becomes `repaired`).
- It will not merge duplicates. You see every row. `match_key` tells you which ones are the same person.
- It will not follow instructions that someone buried inside the CSV. The intake is treated as data.
- It will not retry forever. A failed call is tried once more, except for an auth failure or a response that ran out of room.
- Cells that start with `=`, `+`, `-`, or `@` are prefixed so a spreadsheet does not treat them as formulas.

### Exit codes

| Code | Meaning |
| --- | --- |
| 0 | Wrote the CSV and the report, or `check` passed. |
| 2 | Bad arguments, missing file, binary file, or the intake is over the batch cap. |
| 3 | `XAI_API_KEY` is not set. |
| 4 | The API failed, or the model did not return readable JSON. |

### Try the samples

`samples/messy-leads.csv` is a fictional export: doubled punctuation, a junk row, a repeated header, the same person three times, and one address that must not be guessed.

`samples/messy-invoices.txt` is five scraps from one cafe: a hedged tax, a paid-in-full bill with tax included, a resale slip with no invoice number, a credit memo, and a coffee invoice where only the delivery was paid.

Both use reserved or obviously fictional contact data. They are not real customers.

### Output you should expect to review

Before anyone is emailed, or before a total is entered in the books, read every row whose `email_status` is `repaired` or `suspect`, and every invoice flag that mentions a hedge, a computed tax, or a missing balance. The HTML report repeats that at the bottom on purpose.

## Copyright

Copyright Metta Thomas (M3TA). All rights reserved. No license is granted by this repository until one is added here. The sample files are fiction for testing the script.
