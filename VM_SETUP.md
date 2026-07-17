# FedEx → QuickBooks Desktop: deployment guide

## Architecture

Everything QuickBooks-related runs **inside the VM**. Your local machine only
drops PDFs into a shared folder.

```
LOCAL MACHINE                          VM (QuickBooks Desktop installed)
─────────────                          ──────────────────────────────────
save FedEx PDF into                    fedex_inbox.py watches C:\FedExBills\inbox
the synced inbox folder  ──(sync)──►     • parses the PDF (pypdf + regex)
                                         • validates against QB lists (qbXML queries)
                                         • enters the bill (BillAdd via QBXMLRP2 COM)
read .report.txt in      ◄──(sync)──   moves PDF + report to done\ or flagged\
done\ / flagged\
```

Why not run the parser locally and only ship qbXML to the VM: the
validation/matching step (does the Item exist? which customer:job contains
project 2510177? is this ref number already entered?) requires live queries
against the open company file, which only works where QuickBooks is
installed. Since the VM must run Python + COM anyway, the parser goes with it
— one deployment instead of two.

## What to install where

**Local machine:** nothing new. You already have the repo; you just save
PDFs into the shared inbox folder.

**Inside the VM:**
1. Python 3.x from python.org — check **"Add python.exe to PATH"** during
   install. Use the default 64-bit installer for QuickBooks 2022+ (which is
   64-bit); for QuickBooks 2021 or older install **32-bit** Python so the
   COM component matches.
2. `pip install pypdf pywin32`
3. Copy the `app\` folder into the VM (e.g. `C:\qb-automation\app`).
4. Create `C:\FedExBills\` — the script auto-creates `inbox`, `done`,
   `flagged` under it on first run.

No SDK download is needed: the `QBXMLRP2.RequestProcessor` COM component the
script uses is installed **with QuickBooks Desktop itself**.

## Sharing the inbox folder with your local machine

Pick one:
- **OneDrive (recommended):** sign into the same OneDrive in the VM and put
  the base folder inside it (run with
  `python fedex_inbox.py --watch 30 --base "C:\Users\<you>\OneDrive\FedExBills"`).
  You drop PDFs locally; they appear in the VM within seconds, and reports
  sync back the same way.
- **RDP drive redirection:** enable local drive sharing in your RDP client
  (mstsc → Local Resources → Drives) and point `--base` at
  `\\tsclient\C\FedExBills`. Works, but only while you're connected.

## First run (one-time QuickBooks authorization)

1. In the VM, open QuickBooks Desktop and the company file, logged in as
   **Admin**.
2. Run `python fedex_inbox.py` once from a terminal (put a test PDF in the
   inbox first).
3. QuickBooks pops an "application requesting access" certificate dialog —
   choose **"Yes, always; allow access even if QuickBooks is not running"**
   and confirm. This is remembered; subsequent runs are silent.
4. The script prints the detected product + supported qbXML versions
   (e.g. `QuickBooks Desktop Pro 2023 (qbXML ... 13.0 ...)`) — confirm 13.0
   is listed.

## Day-to-day operation

- Start the watcher in the VM: double-click `app\run_fedex_inbox.bat`
  (or set it up as a Windows Task Scheduler job at logon).
- Drop FedEx PDFs into `inbox\` from your local machine.
- **Bill entered** → PDF + `.report.txt` land in `done\`.
- **Anything wrong** (unknown Item, no matching customer:job, ambiguous
  project number, duplicate ref number, parse total mismatch) → nothing is
  entered, PDF + report land in `flagged\`. Fix the issue in QuickBooks
  (e.g. create the job), then move the PDF back into `inbox\`.
- Nothing is ever auto-created in QuickBooks lists, and a flagged invoice is
  held entirely — no partial bills.

## Version compatibility notes

- qbXML spec 13.0 (what the script requests) is supported by QuickBooks
  Desktop **2013 and later**, including Canadian editions. The
  `SalesTaxCodeRef` per-line tax codes the script sets are a
  Canadian/UK-edition feature — correct for this company file (Canadian, CAD).
- QuickBooks 2022+ is 64-bit → 64-bit Python. 2021 and older → 32-bit Python.
- IIF import is deliberately **not** used: IIF auto-creates any list entry it
  doesn't recognize (the opposite of skip-and-flag), gives no per-line error
  reporting, and bypasses all validation.
