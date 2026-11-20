# Glossary

> Describes Katana at commit `c18dc2c3a4c36704e7e7ae054ab30538e619b889` of its source repository,
> 2026-11-19. Katana keeps one word for each concept; the words here are the ones its code and
> documents use (`docs/GLOSSARY.md`).

## Katana's words

| Word | Means |
| --- | --- |
| **program** | one COBOL compilation unit |
| **job** | a JCL job made of steps |
| **step** | one program execution within a job, with its datasets |
| **profile** | the compile options a result holds for: compiler and version, IBM's TRUNC, ARITH, NUMPROC, code page and others, and the matching GnuCOBOL flags; identified by a SHA-256 hash of its contents |
| **dialect** | the COBOL a profile reads: IBM Enterprise COBOL (the target) or GnuCOBOL 3.1.2 |
| **Semantic IR** | Katana's exact record of what a program does: its data, its paragraphs, its statements, its files |
| **lowering** | the step that writes Java from the Semantic IR |
| **runtime** | `rt/`, the dependency-free Java library the generated programs run on |
| **finding** | something in the input that is wrong, or that Katana cannot make exact; always reported, before any Java is written |
| **fixture** | a description of a job as Katana runs it: its steps, its datasets and their contents, the date and time it runs at, and the profile |
| **port** | a Java program someone else wrote from a COBOL program, which `gradle prove` checks against the original |
| **discovery** | a defect Katana's checks prove in another tool (so far, GnuCOBOL), recorded with the authority it contradicts and a reproducer |
| **golden** | a recorded expected result that a test holds the code to |
| **floor** | a count a test must not fall below, raised by hand as results improve and never lowered by a re-run |
| **Ghost** | the maintainer |

## Findings

The codes Katana reports (`docs/FINDINGS.md`):

| Code | Kind |
| --- | --- |
| `NE-01` | TRUNC(OPT): a binary item's value past its PICTURE depends on IBM's optimizer |
| `NE-02` | overlapping PERFORM ranges |
| `NE-03` | recursive PERFORM |
| `NE-04` | division without ON SIZE ERROR (division by zero is undefined) |
| `NE-05` | OCCURS DEPENDING ON data beyond the count |
| `NE-06` | COMP-1 or COMP-2 (IBM hexadecimal floating point) |
| `NE-08` | arithmetic operands that share storage |
| `NE-09` | a profile option Katana has no mode for (TRUNC(BIN), NUMPROC(PFD)) |
| `NE-10` | an intermediate result that may lose high-order digits |
| `NE-11` | more digits than the ARITH option allows |
| `NE-12` | an 88-level value that does not fit its item |
| `NE-13` | a numeric literal as an edited item's VALUE |
| `HY-01` | GO TO out of a PERFORM range: exact under GnuCOBOL's reading, not yet confirmed on IBM |
| `HY-02` | a statement computed in floating point: the same |
| `CO-01` | source excluded by a false `>>IF`, `>>ELIF` or `>>ELSE`, so not checked |

`NE` is "not exact": IBM's behaviour is undefined, or cannot be reproduced in Java. `HY` is "valid
under hypothesis": listed apart until a mainframe run confirms it. `NE-07` and `HY-03` are retired
numbers, never reused.

## Census stages

How far a GnuCOBOL testsuite case gets through Katana (`docs/when/census.md`):

| Stage | Means |
| --- | --- |
| PASS | the case's whole expected check is met: output, exit status, and error output where expected |
| FLAGPASS | passes once a compile flag Katana does not support is set aside |
| FLAG | uses a compile flag Katana does not support, and does not pass without it |
| FAIL | runs, and does not meet the expected check; each is filed with a reason |
| XFAIL | (GnuCOBOL dialect only) the testsuite itself marks the case, GnuCOBOL 3.1.2 does not meet it either, and Katana agrees with GnuCOBOL |
| NORUN | the run stops: it reaches a finding, abends or times out |
| NOJAVAC | the generated Java does not compile |
| NOLOWER | the Java writer fails |
| REFUSED | the Semantic IR refuses the program |
| NOPARSE | the source is not valid in this dialect |
| UNREAD | met the check, but the check's command could not be fully read; never counted as PASS |
| NOSOURCE | the case has no program source |

## COBOL and mainframe terms

| Word | Means |
| --- | --- |
| **cobc** | GnuCOBOL's compiler; 3.1.2 in Katana's image |
| **copybook** | a COBOL source file included with COPY, typically a record layout |
| **DD** | a step's dataset definition; for a converted program, an environment variable `DD_<name>` holding a file path |
| **PARM** | a step's parameter string, given to the program as z/OS gives it |
| **SYSIN** | a step's input stream; standard input for a converted program |
| **RETURN-CODE** | the value a step ends with; the converted program's exit code |
| **abend** | an abnormal end, reported as `CEE3ABD abend Unnnn` |
| **KSDS** | a VSAM key-sequenced (indexed) dataset |
| **TRUNC, ARITH, NUMPROC, DISPSIGN, DYNAM** | IBM Enterprise COBOL compile options that change what a program computes or how it behaves |
| **NIST CCVS85** | the public COBOL-85 conformance test suite |
| **CardDemo** | AWS's open-source mainframe card-management application, whose batch jobs Katana converts and checks |
