# Katana at a glance

> Describes Katana at commit `843076771d751f4013dbd72a41cd3cf96796566c` of its (private) source
> repository, 2026-11-20. Every figure below names the file at that commit it was read from.

## What it does

Katana is a compiler from COBOL batch programs to Java, written in Java 21. Given a COBOL program, its
copybooks and the compile options it was built with on the mainframe, it writes a Java program that
produces **the same output files, byte for byte,** as the original on the same input data.

- **Deterministic and mechanical.** No machine-learning model runs at conversion time. The same
  program and options always give the same Java.
- **Exact or explicit.** Where a COBOL construct's behaviour cannot be reproduced exactly (because
  IBM leaves it undefined, or it depends on something Java cannot reproduce), Katana says so as a
  **finding**, before any Java is written. It never quietly writes Java that might differ.
- **Checked by running it.** Katana runs the original COBOL and the Java on the same job and compares
  the files each one writes. A result is reported per job and per dataset, with the paragraphs of
  each program the job's data reached.
- **Supported today: GnuCOBOL 3.1.2.** Katana's output is checked against GnuCOBOL's own compiler,
  `cobc`. **IBM Enterprise COBOL is the goal, not yet supported:** Katana has an IBM profile and reads
  IBM's manuals for its rules, but no IBM compiler or mainframe output is in the loop yet, so its IBM
  results are measured only against GnuCOBOL's emulation of IBM's dialect (see "Limits").

Katana is maintained by Ghost (GitHub: ghost-in-the-kernel). The source is private.

## Why it is hard

COBOL's results depend on details that a straightforward translation to Java gets wrong, and gets
wrong silently. Katana's roadmap lists the ones the Java must reproduce exactly
(`docs/ROADMAP.md`, fact F-07):

- **Storage is bytes.** A numeric field is stored in a declared number of digits and truncated to
  fit; REDEFINES lets two descriptions share the same bytes; a group MOVE is a byte copy.
- **Arithmetic follows IBM's rules for intermediate results**, not exact maths. For example,
  `1 / 3 * 300` stored into a field with three integer and two decimal places gives **99.00** on IBM,
  not the 99.99 exact arithmetic gives, because the intermediate quotient keeps a limited number of
  decimal places.
- **Compile options change meaning.** TRUNC, ARITH, NUMPROC and the code page change what a program
  computes, so the same source can have different correct outputs. Katana takes them as an input and
  records them in everything it produces.
- **Signs, encodings and collation** differ by code page (EBCDIC on the mainframe).
- **Control flow** (PERFORM ranges, GO TO, fall-through between paragraphs) and **I/O** (file status
  codes rather than exceptions) have no direct Java equivalent.
- **Some behaviour is undefined,** such as dividing by zero without ON SIZE ERROR, or reading data past
  an OCCURS DEPENDING ON count. No translation can make these exact. Katana reports them instead.

## What it has achieved, measured

All figures are at commit `8430767` and are read from files committed in the source repository.
Every comparison is against GnuCOBOL 3.1.2 (`cobc`). The CardDemo and NIST results were run with
Katana's IBM profile against `cobc`'s IBM-compatibility mode (`-std=ibm`): they show agreement with
GnuCOBOL, and are the groundwork for IBM support, not evidence of it. No IBM mainframe output has
been compared yet (see "Limits" below).

### Real batch jobs: AWS CardDemo

AWS CardDemo is an open-source (Apache-2.0) mainframe credit-card application. Its batch jobs were
converted and run end to end, with every output dataset compared against `cobc -std=ibm`:

| Job | Program(s) | Output datasets compared | Result | Paragraphs reached |
| --- | --- | --- | --- | --- |
| POSTTRAN (post transactions) | CBTRN02C | 6 | all byte-identical | 24 of 26 |
| TRANREPT (transaction report) | CBTRN03C | 6 | all byte-identical | 24 of 26 |
| INTCALC (interest calculation) | CBACT04C | 5 | all byte-identical | 20 of 22 |
| CREASTMT (statements) | CBSTM03A, CBSTM03B | 6 | all byte-identical | 24 of 25; 14 of 14 |

Sources: `oracle/proof/POSTTRAN.v1.report.md`, `TRANREPT.v1.report.md`, `INTCALC.v1.report.md`,
`CREASTMT.v1.report.md`, `CARDDEMO.coverage.md`. Across these jobs the data reached **106 of the
programs' 113 paragraphs**. The paragraphs not reached are listed by name in each report, mostly
abend and error-display routines, and count as unproven. One caveat: CREASTMT's CBSTM03A reads z/OS
control blocks directly from storage, which GnuCOBOL cannot run, so for that step Katana's own Java is
the only run and its result is not a comparison (`CREASTMT.v1.report.md`).

### The NIST COBOL-85 test suite

The NIST CCVS85 suite is the public conformance suite for COBOL-85. Katana runs it under GnuCOBOL and
as converted Java, and checks each program's own reported results. Katana's tracking file gives the
Java side as **"389 runs equal to cobc's today"** (`TODO.md`, item 7.8). The programs whose IBM result
differs from GnuCOBOL's because of a known GnuCOBOL defect are listed by name with that defect
(`test/katana/lower/NistJavaLaneIntegrationTest.java`).

### The GnuCOBOL testsuite census

GnuCOBOL's own testsuite has 378 runnable cases with exact expected output. Katana runs every one
through its full pipeline (parse, convert to Java, compile, run, compare the output, exit status and
error output) and records how far each case gets:

| Census | PASS | also passes without an unsupported flag | FAIL | Source |
| --- | --- | --- | --- | --- |
| GnuCOBOL dialect (supported) | **125** | 16 | 5 | `test/katana/rt/autotest/lowered-gnucobol.golden` |
| IBM dialect (in development; compared with GnuCOBOL, not IBM) | **67** | 10 | 15 | `test/katana/rt/autotest/lowered.golden` |

Most of the testsuite is not IBM COBOL: Katana classes the cases as 70 IBM, 299 other and 9 unknown
(the same files). The rest of the cases stop at an earlier stage, by design: they use syntax the
dialect does not have, or reach a construct Katana reports as a finding rather than guess. Each
stage is defined in the [glossary](glossary.md#census-stages). The PASS counts are floors: they are
raised by hand as cases move, and a test fails if they drop (`test/katana/rt/autotest/floors.txt`).

### Defects found in other tools

Katana's checks have recorded **55 open defects in GnuCOBOL 3.1.2**, almost all in its IBM
dialect: places where its output disagrees with what IBM's manuals document. Each has the authority
quoted, the test that shows it, and a COBOL reproducer (`docs/DISCOVERIES.md`, rows D-001 to D-056,
of which one is withdrawn). Where GnuCOBOL and IBM's manual disagree, Katana follows the manual.

### Findings instead of silent output

Katana's findings are listed in `docs/FINDINGS.md` and kept equal to the code by a test:

- **13 "not exact" codes** (`NE-01` to `NE-13`, no `NE-07`): for example TRUNC(OPT), recursive or
  overlapping PERFORM ranges, division without ON SIZE ERROR, COMP-1/COMP-2 hexadecimal floating
  point, numbers wider than the ARITH option allows.
- **2 "valid under hypothesis" codes** (`HY-01`, `HY-02`): exact under GnuCOBOL's reading of IBM,
  not yet confirmed on a mainframe; listed apart in every report.
- **1 "compiled out" code** (`CO-01`): source excluded by a conditional-compilation directive, so the
  report says that text was not checked.

## Engineering practice

From the source repository at the same commit:

- **Offline, vendored build.** Every dependency is a file in the repository; the build declares no
  package repository and runs with the network off, in a pinned Docker image (`build.gradle`,
  `Dockerfile`).
- **The runtime ships with no dependencies.** The library the generated Java runs on imports nothing
  but the JDK; a test fails if it imports anything else (`docs/modules.json`, rule MOD-02).
- **Documents held to the code.** The contracts (the intermediate representation, the findings, the
  profiles) are documents with numbered rules, and a test fails when a document and the code
  disagree.
- **Mutation testing** with PIT, module by module, with score floors that are only ever raised
  (`scripts/mutation.sh`, `docs/mutation/FLOORS.txt`).
- **A recorded decision log** (`theGhost.md`): each rule in the code cites the decision it carries out.

## Limits, stated plainly

- **IBM Enterprise COBOL is not supported yet.** Every result above is "the same as GnuCOBOL 3.1.2
  gives", including those run under Katana's IBM profile. The count of census cases checked against IBM's own output is 0 at this commit
  (`test/katana/rt/autotest/floors.txt`, `IBMPASS 0`). Reconciling with a customer's z/OS output is a
  planned step (`TODO.md` 9.5).
- **A result covers only the data it was run on**, under the compile options and the clock it pinned,
  and only the paragraphs that data reached.
- **The command line is new and partial.** `katana check`, `build`, `run` and `help` exist; `katana
  prove`, which will check a run's output from the command line, is not on `main` yet (`TODO.md` 8.1).
  The [user guide](user-guide.md) lists what can be run today.
- **Not every class of undefined behaviour is detected yet:** reads of uninitialized storage are not
  yet found (`docs/FINDINGS.md`). Reports state the assumption made instead.

## What is available under NDA

These pages show what Katana does and what it has measured. More is available to a party under a
non-disclosure agreement:

- **The design:** how Katana makes and checks its output, in the internal documentation that these
  pages leave out.
- **The evidence behind each figure:** the full per-job reports, the census results case by case, and
  the source repository the figures are read from.
- **A live demonstration:** `katana check`, `build` and `run` on CardDemo and on test programs, with
  the output compared as it runs.
- **A trial on your own COBOL:** a batch program of yours converted and checked, under an evaluation
  licence; your code and data stay with you and never enter Katana's repository.
- **The roadmap:** the route to IBM Enterprise COBOL support, and what comes after the command line.

## Read next

- [User guide](user-guide.md): installing, running, every command with examples and exit codes.
- [Architecture](architecture.md): the modules and their jobs.
- [Glossary](glossary.md).
