# User guide

> Describes Katana at commit `843076771d751f4013dbd72a41cd3cf96796566c` of its source repository,
> 2026-11-20. Paths in backticks are files in that repository.

## What you can run at this commit

| What | For |
| --- | --- |
| [`katana check`, `build`, `run`, `help`](#the-katana-command) | converting a COBOL program to Java, checking what Katana would report, and running the result |
| [`gradle prove`](#gradle-prove-check-a-java-translation-against-its-cobol) | checking a Java translation of a COBOL program (anyone's, not only Katana's) against the original; the planned `katana prove` |
| [Running a converted program](#running-a-converted-program) | running generated Java directly, with its files, options and exit codes |
| [`katana.rt.Ksds`](#katanartksds-load-or-unload-an-indexed-file) | loading or unloading an indexed (VSAM KSDS) file for the converted programs |
| [Build and test commands](#build-and-test-commands) | building Katana and running its test suites |

Planned and not yet on `main`: `katana prove` (equivalence on a run's data, with a printable report),
then `fixture`, `pass`, `report` and `pack`, with `--json` on every report (`docs/ROADMAP.md` §8.1,
`TODO.md` 8.1 to 8.4).

## Install

Katana builds and runs inside a pinned Docker image, defined by `Dockerfile`: Gradle 8.14.3 on JDK 21,
with GnuCOBOL 3.1.2 installed. The build refuses any other Gradle version (`build.gradle`).

You need Docker and a checkout of the Katana repository (it is private; access is by invitation).

```sh
cd katana
docker build -t katana .
```

Every command then runs in the image, with the checkout mounted, the network off, as your own user:

```sh
docker run --rm --network none -u "$(id -u)" -v "$PWD":/katana katana gradle test --offline
```

`--offline` is required: the build has every dependency in the repository (`third_party/`) and
declares no package repository, so it never reaches the network.

The test suites that run COBOL also need the test corpus, fetched once on the host (it needs the
network):

```sh
corpus/fetch.sh                                   # every corpus listed in corpus/MANIFEST.md
corpus/fetch.sh nist-cobol85-test-suite carddemo  # only those
corpus/fetch.sh gnucobol-autotests                # GnuCOBOL's testsuite cases, only when named
```

The corpus is cloned into `corpus/` and never committed; its licences are in `corpus/MANIFEST.md`.

---

## The `katana` command

The command-line tool, from TODO 8.1's first two slices (decision R-112; `katana`, `cli/Main.java`,
`cli/Report.java`, `project/`). Its help text and the README's section "The katana command" are held
equal by a test (`test/katana/cli/MainTest.java`). `katana prove` and the other commands the roadmap
plans (`fixture`, `pass`, `report`, `pack`) are not on `main` yet.

### Setting it up

Build Katana's classes and the runtime's once, in the image:

```sh
docker run --rm --network none -u "$(id -u)" -v "$PWD":/katana katana gradle classes rtClasses --offline
```

Then run `./katana` from the repository root. The script runs `java` directly when a Java 21 is on
the `PATH`; otherwise it runs inside the pinned image with no network, the current folder mounted at
its own path, so files must be named under the current folder (`katana`, its header). If the classes
are not built it says so and exits 3. It writes only under `--out`.

### Commands

```
katana check <file> [--profile ibm|gnucobol|PATH] [--copy DIR]... [--strict]
katana build <file> [--profile ibm|gnucobol|PATH] [--copy DIR]... [--out DIR] [--strict] [--compile]
katana run   <file> [--profile ibm|gnucobol|PATH] [--copy DIR]... [--out DIR] [--strict] [-- program args...]
katana help
```

| Command | What it does |
| --- | --- |
| `check` | parses, builds and converts the file in memory; prints the findings; writes nothing |
| `build` | writes `<Class>.java` for each program in the file under `--out`, and prints the findings; with `--compile`, also compiles them with `javac` against the runtime alone, into the same folder. The Java is the direct translation, each statement with its COBOL line; no refactoring is applied (`project/Build.java`) |
| `run` | builds, compiles, and runs the program (the file's first program) in a separate JVM. Its standard input, output and error pass through unchanged, and its exit code is `katana`'s. Everything `katana` itself says (the report, a one-line header, a one-line footer with the exit code) goes to standard error, so the program's standard output stays clean |
| `help` (also `--help`, `-h`) | prints the usage |

| Option | Meaning |
| --- | --- |
| `--profile P` | the dialect profile: `ibm` (the default), `gnucobol`, or the path of a profile file |
| `--copy DIR` | a copybook folder, searched after the source file's own folder, in the order given; repeatable |
| `--out DIR` | where `build` and `run` write (default `katana-out`); nothing is written anywhere else |
| `--compile` | `build` only: also compile the Java, as `run` does |
| `--strict` | any finding refuses the program: no Java is written, exit 1 |
| `--trace` | an internal fault prints its stack trace |
| `--` | `run` only: everything after it is passed to the program as its arguments (the first is its PARM) |

One file per command. An unknown option, an option given twice (except `--copy`), a missing value,
or no file is a usage error.

**What it prints** (`cli/Report.java`): a first line naming the command, the source, the profile and
its hash, and the copybook folders searched. Then either `refused, no Java written:` with the
reasons (a missing copybook names the folders searched and suggests `--copy`), or one line per class
(`wrote` or `would write`, with its statement count), then the findings: each coded finding (`NE-`,
`HY-`, `CO-`) with its place and detail, other findings as written, and once per code what it means.
`no findings` when there are none.

**Exit codes** (`cli/Main.java`, R-112):

| Code | Meaning |
| --- | --- |
| 0 | Java written (for `check`: it would be); any findings printed. Findings alone do not change the exit code unless `--strict` is given |
| 1 | refused: no Java written (the program cannot be built, a copybook is missing, or a finding under `--strict`) |
| 2 | usage error, or the file cannot be read; the usage is printed to standard error |
| 3 | an internal fault, including `javac` refusing Java that Katana wrote (Katana's fault, not the program's), or `./katana` run before the classes are built |
| (run) | once the program has run, its own exit code (see [the exit codes of a converted program](#running-a-converted-program)); the footer on standard error says so |

**Running a program.** `run` passes the profile's DISPLAY sign form to the program
(`-Dkatana.dispsign`; under `--profile gnucobol` always `SEP`, as GnuCOBOL shows it;
`project/Build.java`). The program inherits `katana`'s environment when `katana` runs on a local
Java, so its files can be given as `DD_<name>` variables. When the script falls back to the Docker
image it passes no environment variables into the container (the `katana` script has no `-e`), so in
that case only standard input, arguments, and files named in the current folder reach the program.

### Examples

Not executed for this page; written from the code and the help text. Using a program and copybook
from Katana's own repository (`oracle/fixture/trivial/`):

```sh
./katana check oracle/fixture/trivial/cbl/COPYREC.cbl --copy oracle/fixture/trivial/cpy
./katana build oracle/fixture/trivial/cbl/COPYREC.cbl --copy oracle/fixture/trivial/cpy --out build/copyrec --compile
./katana build legacy/PAYROLL.cbl --strict ; echo "exit $?"   # 1 if any finding
./katana run   legacy/REPORT.cbl --profile gnucobol < input.txt > report.txt
./katana run   legacy/PARMPGM.cbl -- "RUN=2024-03-01"
```

(`legacy/...` names are illustrative.)

---

## `gradle prove`: check a Java translation against its COBOL

Runs a COBOL program and someone's Java translation of it (a **port**) on the same job, and reports,
for each output dataset and each step's printed output, whether the two are the same. The roadmap
names this the standalone harness that "can verify anyone's Java" (`docs/ROADMAP.md` §0); `katana prove`,
not yet on `main`, is to replace it (`build.gradle`, the `prove` task's comment).

**Usage** (`build.gradle` task `prove`; `oracle/prove/Prove.java`):

```sh
gradle prove --offline --args="<fixture> <PGM>=<command...>"
```

| Argument | Meaning |
| --- | --- |
| `<fixture>` | a fixture file: the job to run (its steps, the input datasets and their contents, the output datasets and their record layouts, a pinned date and time). Its folder must contain `cbl/` (the COBOL source) and `cpy/` (the copybooks). The format is `docs/FIXTURE.md` in the repository. |
| `<PGM>=<command...>` | the step's program name, `=`, then the command that runs the port. Any further arguments are added to the command. |

Run it from the repository root: a fixture's `from=repo:...` paths are read relative to it. Each run
has a two-minute time limit. Work files go under `build/prove/`.

**The port's contract** (`docs/LANES.md`, LANE-01 to LANE-04): the port reads and writes its files
through environment variables, one per DD, `DD_<name>` holding a file path; its one argument is the
PARM; SYSIN is its standard input; its standard output is the step's printed output; its exit code is
the step's return code. Sequential records are fixed length, back to back, with no line terminators.

**Output,** on standard output:

```
prove ACCRUE: the port against cobc -std=ibm, the oracle (R-013)
SAME   KATANA.FOREIGN.OUT
SAME   stdout of ACCRUE
```

One `SAME` or `DIFFER` line per output dataset (a `DIFFER` is followed by indented lines describing
the difference), then one per step's printed output. A port
that leaves an output dataset empty or unwritten that the COBOL wrote is `REFUSED`, with the reason,
and nothing is compared.

**Exit codes:**

| Code | Meaning |
| --- | --- |
| 0 | the port matches: nothing refused, every dataset and every printed output the same |
| 1 | refused, or something differs |

Gradle reports a failed build on exit code 1. The arguments are not validated: run without them, it
ends with an exception rather than a usage message.

**Example,** using the repository's own trial fixture, `oracle/fixture/foreign/ACCRUE.fixture` (one
step, `ACCRUE`, which reads an account file and writes one with interest accrued), and a port compiled
into `build/port` as a class `Accrue` (the port's name and folder are your own):

```sh
docker run --rm --network none -u "$(id -u)" -v "$PWD":/katana katana \
  gradle prove --offline --args="oracle/fixture/foreign/ACCRUE.fixture ACCRUE=java -cp build/port Accrue"
```

---

## Running a converted program

Each COBOL program Katana converts becomes one Java class with a `main` method
(`lower/Lower.java`). It runs on Katana's runtime library, `rt/`, which has no dependencies beyond the
JDK. Build the runtime with Gradle (its classes land in `build/classes/java/rt`) and put it on the
class path.

**Inputs** (`docs/LANES.md`; `rt/katana/rt/SeqFile.java`, `KeyedFile.java`):

| Input | How it is given |
| --- | --- |
| files | a file the program ASSIGNs to `X` is the path in environment variable `DD_X` (then `dd_X`), else a file named `X` in the working directory |
| PARM | the first command-line argument, for a program with PROCEDURE DIVISION USING; passed as z/OS passes it |
| SYSIN | standard input |
| date and time | `COB_CURRENT_DATE` (for example `2024-03-01 12:00:00.00 +00:00`) and `TZ`, so a run can be repeated exactly |
| paragraph trace | `KATANA_TRACE_FILE`: a file listing each section and paragraph the program enters |

**Options,** as Java system properties (`-D...`):

| Property | Default | Meaning |
| --- | --- | --- |
| `katana.codepage` | `COBC_ASCII` | the character set of the program's data, by `rt/Codepage` name |
| `katana.dispsign` | as converted | `COMPAT` or `SEP`: how DISPLAY shows a signed number (IBM's DISPSIGN option) |
| `katana.upsi` | `00000000` | the eight UPSI switches |
| `katana.dynam` | `NODYNAM` | `DYNAM` for IBM's DYNAM option: how a CALL by literal finds its program |
| `katana.debug` | `N` | `Y` turns on USE FOR DEBUGGING sections (IBM's DEBUG run-time option) |

**Exit codes** (`lower/Lower.java`; `docs/LANES.md` LANE-04, LANE-08, LANE-17):

| How the program ends | Exit code | On standard error |
| --- | --- | --- |
| normally (STOP RUN, GOBACK) | RETURN-CODE modulo 4096; 0 if never set | nothing |
| abend (for example a CALL to `CEE3ABD`) | the abend code | `CEE3ABD abend Unnnn` and the reason |
| reaches a construct Katana reported as a finding | 1 (an uncaught exception) | the exception |

**Example,** a program converted to class `COPYREC` in `build/out`, with one input and one output file
(illustrative names):

```sh
DD_INFILE=in.dat DD_OUTFILE=out.dat COB_CURRENT_DATE="2024-03-01 12:00:00.00 +00:00" TZ=UTC \
  java -cp build/classes/java/rt:build/out COPYREC
echo "return code $?"
```

---

## `katana.rt.Ksds`: load or unload an indexed file

Converted programs keep indexed files in the runtime's own format. This tool loads records into one,
or unloads them in key order (`rt/katana/rt/Ksds.java`):

```sh
java -cp build/classes/java/rt katana.rt.Ksds load|unload LRECL PRIME ALTERNATES SEQUENTIAL INDEXED
```

| Argument | Meaning |
| --- | --- |
| `load` or `unload` | from the sequential file into the indexed one, or out of it in key order |
| `LRECL` | the record length |
| `PRIME` | the primary key as `length,offset` |
| `ALTERNATES` | `-` for none, or each alternate key as `length,offset,u` (unique) or `length,offset,d` (duplicates allowed), separated by `;` |
| `SEQUENTIAL`, `INDEXED` | the two files' paths |

Example: a 300-byte record keyed on its first 11 bytes, with no alternate keys:

```sh
java -cp build/classes/java/rt katana.rt.Ksds load 300 11,0 - acct.seq acct.idx
```

**Exit codes:** 0 on success; 12, with the reason on standard error, on failure.

---

## Build and test commands

All run in the image, as in [Install](#install). Gradle's exit code is non-zero on any failure.

| Command | What it does |
| --- | --- |
| `gradle test --offline` | the fast suite: unit tests and the tests that hold the documents to the code. Needs no GnuCOBOL and no corpus. |
| `gradle integration --offline` | the suites that need GnuCOBOL and the corpus: NIST, the GnuCOBOL testsuite census, CardDemo. Long: send the output to a file. |
| `gradle integration --offline --tests 'katana.oracle.nist.*'` | one suite only |
| `gradle build --offline` | compile and run the fast suite (the image's default command, and CI's) |

Options for `gradle integration` (`build.gradle`):

| Option | Meaning |
| --- | --- |
| `-Dkatana.std=ibm` | run NIST in one dialect only (default: both `cobol85` and `ibm`) |
| `-Dkatana.nist.modules=NC,IF` | run only those NIST modules |
| `-Dkatana.record=true` | rewrite the recorded expected results from this run, for review as a diff |

A test task that runs no tests fails rather than passing silently (`build.gradle`).

For people changing Katana itself, `scripts/prepush.sh` is the check before a push: it runs, in the
image with the network off on a fresh clone, the suites the change touches (`--all` for every suite,
`--list-suites PATH...` to see which a path selects), and ends with the line
`pre-push check: passed` or `pre-push check: FAILED`; it exits 0 when passed. Mutation testing is
`scripts/mutation.sh <module>`, which exits 1 when a module's score falls below its recorded floor.
