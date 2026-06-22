# FIBERNET — Telecommunication / Broadband Billing System
### A COBOL on IBM i (AS/400) case study

FIBERNET is a small but complete broadband-service back-office system written
in **ILE COBOL** with **DDS** database, display and printer files for **IBM i
(AS/400 / iSeries)**. It is delivered as **separate modules**, one source
member per program, so each module can be read, compiled and demonstrated on
its own.

---

## 1. What the system does

| Area | Module | Function |
|------|--------|----------|
| Menu | `FBNMENU` | Main menu; calls every other module |
| Masters | `CUSTMNT` | Customer master — Add / Change / Delete / Inquiry |
| Masters | `PLANMNT` | Broadband plan master — Add / Change / Delete / Inquiry |
| Billing | `BILLGEN` | Batch: raise monthly bills for active customers (rental + 18% GST) |
| Billing | `PAYPROC` | Post a payment against a bill; update paid amount & status |
| Service | `CMPMNT` | Log / update / inquire customer complaints |
| Output | `BILLRPT` | Printed report of outstanding (unpaid / part-paid) bills |

---

## 2. Folder ↔ IBM i source-file mapping

On IBM i, source lives in *source physical files* (members), not in the IFS.
The folders here map one-to-one onto those source files:

| Folder in this zip | IBM i source physical file | Members (type) |
|--------------------|----------------------------|----------------|
| `QDDSSRC/`   | `FIBERNET/QDDSSRC`   | `*.PF`, `*.LF`, `*.DSPF`, `*.PRTF` |
| `QCBLLESRC/` | `FIBERNET/QCBLLESRC` | `*.CBLLE` (ILE COBOL) |
| `QCLSRC/`    | `FIBERNET/QCLSRC`    | `*.CLLE` (ILE CL) |
| `docs/`      | —                    | this README |

The file extensions are only there to make the members readable off-platform.
On IBM i a member has a *name* and a *type*; e.g. `CUSTMAST.PF` becomes member
`CUSTMAST` of type `PF`, `CUSTMNT.CBLLE` becomes member `CUSTMNT` type `CBLLE`.

---

## 3. Data files (DDS)

| File | Type | Key | Purpose |
|------|------|-----|---------|
| `CTLFILE`   | PF | `CTLID` | Single control record holding the next customer / bill / payment / complaint numbers |
| `PLANMAST`  | PF | `PLANID` | Broadband plan catalogue (speed, monthly rental, data limit) |
| `CUSTMAST`  | PF | `CUSTID` | Customer master (name, address, subscribed plan `CPLANID`, status) |
| `BILLMAST`  | PF | `BILLNO` | One record per monthly bill (rental, tax, total, paid, status) |
| `BILLMAST1` | LF | `BCUSTID + BILLYM` | Logical view of `BILLMAST` ordered by customer + period (used by the report) |
| `PAYMAST`   | PF | `PAYNO` | One record per payment received |
| `CMPMAST`   | PF | `CMPNO` | Customer complaints / service requests |

### Design conventions
* **Unique field prefixes per file** avoid duplicate-name clashes when several
  files are copied into one program (`BILLMAST` uses `BCUSTID`, `PAYMAST` uses
  `PCUSTID`, `CMPMAST` uses `CCUSTID`, and the customer's plan is `CPLANID` so
  it never collides with `PLANMAST`'s `PLANID`).
* **Screen detail fields share names with the database record**, so a program
  can move a whole record with one `MOVE CORRESPONDING`. The few fields that
  exist in two formats are always qualified (`CUSTID OF CUSTREC`).
* **Dates are stored as 8-digit numbers `YYYYMMDD`** (and periods as `YYYYMM`),
  which keeps the COBOL simple and portable. Today's date is obtained with
  `FUNCTION CURRENT-DATE`.
* **GST / tax = 18%** of the plan rental. **Bill status**: `O`=open,
  `R`=part-paid, `P`=fully paid.

---

## 4. Getting the source onto IBM i

1. Create the library and source files:
   ```
   CRTLIB FIBERNET
   CRTSRCPF FILE(FIBERNET/QDDSSRC)   RCDLEN(112)
   CRTSRCPF FILE(FIBERNET/QCBLLESRC) RCDLEN(112)
   CRTSRCPF FILE(FIBERNET/QCLSRC)    RCDLEN(112)
   ```
2. Upload each file in this zip into the matching member. Any of these work:
   * **IBM i ACS → "Upload"** into the source files, or
   * FTP in ASCII mode to `/QSYS.LIB/FIBERNET.LIB/QDDSSRC.FILE/CUSTMAST.MBR`, or
   * copy from the IFS with
     `CPYFRMSTMF FROMSTMF('/home/me/CUSTMAST.PF')
      TOMBR('/QSYS.LIB/FIBERNET.LIB/QDDSSRC.FILE/CUSTMAST.MBR') MBROPT(*REPLACE)`.

   Member names must match the file name without the extension (e.g. the member
   for `CUSTMNT.CBLLE` is `CUSTMNT`).

---

## 5. Building (compile order matters)

The database, display and printer **objects must exist before the COBOL
programs are compiled**, because each program does `COPY DDS-ALL-FORMATS`,
which reads the file's record format at compile time.

Easiest path — compile and run the supplied CL driver:
```
CRTBNDCL PGM(FIBERNET/BLDFBN) SRCFILE(FIBERNET/QCLSRC) SRCMBR(BLDFBN)
CALL     FIBERNET/BLDFBN
```
`BLDFBN` performs, in order:
1. `CRTPF`  — CTLFILE, PLANMAST, CUSTMAST, BILLMAST, PAYMAST, CMPMAST
2. `CRTLF`  — BILLMAST1 (after BILLMAST)
3. `CRTDSPF`— FBNMENUD, CUSTMNTD, PLANMNTD, PAYPROCD, CMPMNTD
4. `CRTPRTF`— BILLRPTP
5. `CRTBNDCBL` — CUSTMNT, PLANMNT, BILLGEN, PAYPROC, CMPMNT, BILLRPT, FBNMENU
6. Seeds the `CTLFILE` control record.

---

## 6. Running

```
ADDLIBLE FIBERNET
CALL     FIBERNET/FBNMENU
```
Menu options: 1 Customer · 2 Plan · 3 Generate Bills · 4 Payment ·
5 Complaint · 6 Outstanding Report · 90 Sign off.
`F3` exits a program; `F12` cancels the current entry.

A typical first run: add a plan (option 2) → add a customer on that plan
(option 1) → generate bills (option 3) → take a payment (option 4) →
print outstanding bills (option 6, then `WRKSPLF` to view the spooled report).

---

## 7. Note on portability

The code targets **IBM i ILE COBOL (COBOL/400)** and follows its conventions
for externally-described files (`COPY DDS-ALL-FORMATS`), workstation I/O
(`ORGANIZATION IS TRANSACTION`, `INDARA` indicators) and the `DATABASE-`,
`WORKSTATION-…-SI` and `PRINTER-` assignment names. On a different compiler or
an older OPM COBOL/400 you may need minor adjustments (for example the exact
indicator-area wiring or the `COPY DDS` variant). It cannot be compiled off an
IBM i system. Field layouts, business logic and program structure are complete
and ready to compile once the members are on the machine.
