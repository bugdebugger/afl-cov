# afl-cov - AFL Fuzzing Code Coverage

## Introduction
`afl-cov` uses test case files produced by the AFL fuzzer (see:
[http://lcamtuf.coredump.cx/afl/](http://lcamtuf.coredump.cx/afl/) to produce
gcov code coverage results of the targeted binary. Code coverage is
interpreted from one case to the next by `afl-cov` in order to determine which
new functions and lines are hit by AFL with each new test case. Further,
`afl-cov` allows for specific lines or functions to be searched for within
coverage results, and when a match is found the corresponding test case file is
displayed. This allows the user to discover which AFL test case is the first to
exercise a particular function. In addition, `afl-cov` produces a "zero
coverage" report of functions and lines that were never executed during an AFL
fuzzing run.

Although of no use to AFL itself, the main application of `afl-cov` is to wrap
some automation around gcov and thereby provide data on how to maximize code
coverage with AFL fuzzing runs. Manual interpretation of cumulative gcov
results from AFL test cases is usually still required, but the "fiddly" steps
of iterating over all test cases and generating code coverage reports (along
with the "zero coverage" report) is automated by `afl-cov`.

Producing code coverage data for AFL test cases is an important step to try
and maximize code coverage, and thereby help to maximize the effectiveness of
AFL. For example, some binaries have code that is reachable only after a
complicated (or even cryptographic) test is passed, and AFL may not be able to
exercise this code without taking special measures. These measures commonly
include patching the project code to bypass such tests. (For example, there is
a patch to solve this problem for a CRC test in libpng included in the AFL
sources at `experimental/libpng_no_checksum/libpng-nocrc.patch`.)
When a project implements a patch to assist AFL in reaching code that would
otherwise be inaccessible, a natural question to ask is whether the patch is
effective. Code coverage results can help to verify this.

## Prerequisites
`afl-cov` requires the following software:

 * afl-fuzz
 * python
 * gcov, lcov, genhtml

Note that `afl-cov` can parse files created by `afl-fuzz` from a different
system, so technically `afl-fuzz` does not need to be installed on the same
system as `afl-cov`. This supports scenarios where fuzzing output is collected,
say, within a git repository on one system, and coverage results are produced
on a different system. However, most workflows typically focus on producing
`afl-cov` results quickly for current fuzzing runs on the same system.

## Workflow
At a high level, the general workflow for `afl-cov` is:

1. Create a spare copy of the project sources compiled with gcov profiling support.
2. Run `afl-cov` while `afl-fuzz` is building test cases.
3. Review the cumulative code coverage results in the final web report.

Now, in more detail:

* Copy the project sources to two different directories
`/path/to/afl-fuzz-output/` and `/path/to/project-gcov/`. The first
will contain the project binaries compiled for AFL fuzzing, and the second will
contain the project binaries compiled for gcov profiling support
(gcc `-fprofile-arcs -ftest-coverage`).

* Start up `afl-cov` in `--live` mode before also starting the `afl-fuzz`
fuzzing cycle. The command line arguments to `afl-cov` must specify the path to
the output directory used by `afl-fuzz`, and the command to execute along with
associated arguments. This command and arguments should closely resemble the
manner in which `afl-fuzz` executes the targeted binary during the fuzzing
cycle. Note that if there is already an existing directory of AFL fuzzing
results, then just omit the `--live` argument to process these results.  Here
is an example:

```bash
$ cd /path/to/project-gcov/
$ afl-cov -d /path/to/afl-fuzz-output/ --live --coverage-cmd \
"cat AFL_FILE | LD_LIBRARY_PATH=./lib/.libs ./bin/.libs/somebin -a -b -c" \
--code-dir .
```

Note the `AFL_FILE` string above refers to the test case file that AFL will
build in the `queue/` directory under `/path/to/project-fuzz`. Just leave this
string as-is - `afl-cov` will automatically substitute it with each AFL
`queue/id:NNNNNN*` in succession as is builds the code coverage reports.

Also, in the above command, this handles the case where the AFL fuzzing cycle
is fuzzing the targeted binary via stdin. This explains the
`cat AFL_FILE | ... ./bin/.lib/somebin ...` invocation. For the other style of
fuzzing with AFL where a file is read from the filesystem, here is an example:

```bash
$ cd /path/to/project-gcov/
$ afl-cov -d /path/to/afl-fuzz-output/ --live --coverage-cmd \
"LD_LIBRARY_PATH=./lib/.libs ./bin/.libs/somebin -f AFL_FILE -a -b -c" \
--code-dir .
```

* With `afl-cov` running, open a separate terminal/shell, and launch
`afl-fuzz`:

```bash
$ cd /path/to/project-fuzzing/
$ LD_LIBRARY_PATH=./lib/.libs afl-fuzz -T somebin -t 1000 -i ./test-cases/ \
-o /path/to/afl-fuzz-output/ ./bin/.libs/somebin -a -b -c
```

The familiar AFL status screen will be displayed, and `afl-cov` will start
generating code coverage data.

![alt text][AFL-status-screen]

[AFL-status-screen]: https://github.com/mrash/afl-cov/raw/master/doc/AFL_status_screen.png "AFL Fuzzing Cycle"

Here is a sample of what the `afl-cov` output looks like:

```bash
$ afl-cov -d /path/to/afl-fuzz-output/ --live --coverage-cmd \
"LD_LIBRARY_PATH=./lib/.libs ./bin/.libs/somebin -f AFL_FILE -a -b -c" --code-dir .
[+] Imported 184 files from: /path/to/afl-fuzz-output/queue
[+] AFL file: id:000000,orig:somestr.start (1 / 184), cycle: 0
    lines......: 18.6% (1122 of 6032 lines)
    functions..: 30.7% (100 of 326 functions)
    branches...: 14.0% (570 of 4065 branches)
[+] AFL file: id:000001,orig:somestr256.start (2 / 184), cycle: 2
    lines......: 18.7% (1127 of 6032 lines)
    functions..: 30.7% (100 of 326 functions)
    branches...: 14.1% (572 of 4065 branches)
[+] Coverage diff id:000000,orig:somestr.start id:000001,orig:somestr256.start
    Src file: /path/to/project-gcov/lib/proj_decode.c
      New 'line' coverage: 140
      New 'line' coverage: 141
      New 'line' coverage: 142
    Src file: /path/to/project-gcov/lib/proj_util.c
      New 'line' coverage: 217
      New 'line' coverage: 218
[+] AFL file: id:000002,orig:somestr384.start (3 / 184), cycle: 10
    lines......: 18.8% (1132 of 6032 lines)
    functions..: 30.7% (100 of 326 functions)
    branches...: 14.1% (574 of 4065 branches)
[+] Coverage diff id:000001,orig:somestr256.start id:000002,orig:somestr384.start
    Src file: /path/to/project-gcov/lib/proj_decode.c
      New 'line' coverage: 145
      New 'line' coverage: 146
      New 'line' coverage: 147
    Src file: /path/to/project-gcov/lib/proj_util.c
      New 'line' coverage: 220
      New 'line' coverage: 221
[+] AFL file: id:000003,orig:somestr.start (4 / 184), cycle: 5
    lines......: 18.9% (1141 of 6032 lines)
    functions..: 31.0% (101 of 326 functions)
    branches...: 14.3% (581 of 4065 branches)
[+] Coverage diff id:000002,orig:somestr384.start id:000003,orig:somestr.start
    Src file: /path/to/project-gcov/lib/proj_message.c
      New 'function' coverage: validate_cmd_msg()
      New 'line' coverage: 244
      New 'line' coverage: 247
      New 'line' coverage: 248
      New 'line' coverage: 250
      New 'line' coverage: 255
      New 'line' coverage: 262
      New 'line' coverage: 263
      New 'line' coverage: 266
.
.
.
[+] Coverage diff id:000182,src:000000,op:havoc,rep:64 id:000184,src:000000,op:havoc,rep:4
[+] Processed 184 / 184 files

[+] Final zero coverage report: /path/to/afl-fuzz-output/cov/zero-cov
[+] Final positive coverage report: /path/to/afl-fuzz-output/cov/pos-cov
[+] Final lcov web report: /path/to/afl-fuzz-output/cov/web/lcov-web-final.html
```

In the last few lines above, the locations of the final web coverage and zero
coverage reports are shown. The zero coverage reports contains function names
that were never executed across the entire `afl-fuzz` run.

The code coverage results in `/path/to/afl-fuzz-output/cov/web/lcov-web-final`
represent cumulative code coverage across all AFL test cases. This data can then
be reviewed to ensure that all expected functions are indeed exercised by AFL -
just point a web browser at `/path/to/afl-fuzz-output/cov/web/lcov-web-final.html`.
Below is a sample of what this report looks like for a cumulative AFL fuzzing
run - this is against the fwknop project, and the full report is
[available here](https://www.cipherdyne.org/fwknop/2.6.7-afl-lcov-results/).
Note that even though fwknop has a dedicated set of
[AFL wrappers](https://github.com/mrash/fwknop/tree/master/test/afl), it is still
difficult to achieve high percentages of code coverage. This provides evidence
that measuring code coverage under AFL fuzzing runs is an important aspect of
trying to achieve maximal fuzzing results. Every branch/line/function that is
not exercised by AFL represents a location for which AFL has not been given the
opportunity to find bugs.

![alt text][AFL-lcov-web-report]

[AFL-lcov-web-report]: https://github.com/mrash/afl-cov/raw/master/doc/AFL_lcov_web_report.png "AFL lcov web report"

### Other Examples
The workflow above is probably the main strategy for using `afl-cov`. However,
additional use cases are supported such as:

1. Suppose there are a set of wrapper scripts around `afl-fuzz` to run fuzzing
cycles against various aspects of a project. By building a set of corresponding
`afl-cov` wrappers, and then using the `--disable-coverage-init` option on all
but the first of these wrappers, it is possible to generate code coverage results
across the entire set of `afl-fuzz` fuzzing runs. (By default, `afl-cov` resets
gcov counters to zero at start time, but the `--disable-coverage-init` stops this
behavior.) The end result is a global picture of code coverage across all
invocations of `afl-fuzz`.

2. Specific functions can be searched for in the code coverage results, and
`afl-cov` will return the first `afl-fuzz` test case where a given function is
executed. This allows `afl-cov` to be used as a validation tool by other scripts
and testing infrastructure. For example, a test case could be written around
whether an important function is executed by `afl-fuzz` to validate a patching
strategy mentioned in the introduction.

Here is an example where the first test case that executes the function
`validate_cmd_msg()` is returned (this is after all `afl-cov` results have been
produced in the main workflow above):

```bash
$ ./afl-cov -d /path/to/afl-fuzz-output --func-search "validate_cmd_msg"
[+] Function 'validate_cmd_mag()' executed by: id:000002,orig:somestr384.start
```

An equivalent way of searching the coverage results is to just `grep` the function
from the `cov/id-delta-cov` file described below. Note the number _"3"_ in the output
below is the AFL cycle number where the function is first executed:

```bash
$ grep validate_cmd_msg /path/to/afl-fuzz-output/cov/id-delta-cov
id:000002,orig:somestr384.start, 3, /path/to/project-gcov/file.c, function, validate_cmd_msg()
```

## Directory / File Structure
`afl-cov` creates a few files and directories for coverage results within the
specified `afl-fuzz` directory (`-d`). These files and directories are
displayed below, and all are contained within the main
`/path/to/afl-fuzz-output/cov/` directory:

 * `cov/diff/` - contains new code coverage results when `queue/id:NNNNNN*` file
                 causes `afl-fuzz` to execute new code.
 * `cov/lcov/` - contains raw code coverage data produced by the lcov front-end to gcov.
 * `cov/web/`  - contains code coverage results in web format produced by `genhtml`.
 * `cov/zero-cov` - file that lists all functions (and optionally lines) that are never
                    executed by any `afl-fuzz` test case.
 * `cov/pos-cov` - file that lists all functions (and optionally lines) that are
                   executed at least once by an `afl-fuzz` test case.
 * `cov/id-delta-cov` - lists the functions (and optionally lines) that are executed by
                        the first `id:000000*` test case, and then lists all new
                        functions/lines executed in subsequent test cases.

## Usage Information
Basic `--help` output appears below:

    usage: afl-cov [-h] [-e COVERAGE_CMD] [-d AFL_FUZZING_DIR] [-c CODE_DIR] [-O]
               [--disable-cmd-redirection] [--disable-lcov-web]
               [--disable-coverage-init] [--coverage-include-lines] [--live]
               [--sleep SLEEP] [--lcov-web-all] [--func-search FUNC_SEARCH]
               [--line-search LINE_SEARCH] [--src-file SRC_FILE]
               [--afl-queue-id-limit AFL_QUEUE_ID_LIMIT] [-v] [-V] [-q]

    optional arguments:
      -h, --help            show this help message and exit
      -e COVERAGE_CMD, --coverage-cmd COVERAGE_CMD
                            set command to exec (including args, and assumes code
                            coverage support)
      -d AFL_FUZZING_DIR, --afl-fuzzing-dir AFL_FUZZING_DIR
                            top level AFL fuzzing directory
      -c CODE_DIR, --code-dir CODE_DIR
                            directory where the code lives (compiled with code
                            coverage support)
      -O, --overwrite       overwrite existing coverage results
      --disable-cmd-redirection
                            disable redirection of command results to /dev/null
      --disable-lcov-web    disable generation of all lcov web code coverage
                            reports
      --disable-coverage-init
                            disable initialization of code coverage counters at
                            afl-cov startup
      --coverage-include-lines
                            include lines in zero-coverage status files
      --live                process a live AFL directory, and afl-cov will exit
                            when it appears afl-fuzz has been stopped
      --sleep SLEEP         In --live mode, # of seconds to sleep between checking
                            for new queue files
      --lcov-web-all        generate lcov web reports for all id:NNNNNN* files
                            instead of just the last one
      --func-search FUNC_SEARCH
                            search for coverage of a specific function
      --line-search LINE_SEARCH
                            search for coverage of a specific line number
                            (requires --src-file)
      --src-file SRC_FILE   restrict function or line search to a specific source
                            file
      --afl-queue-id-limit AFL_QUEUE_ID_LIMIT
                            limit the number of id:NNNNNN* files processed in the
                            AFL queue/ directory
      -v, --verbose         verbose mode
      -V, --version         print version and exit
      -q, --quiet           quiet mode

## License
`afl-cov` is released as open source software under the terms of
the **GNU General Public License (GPL v2)**. The latest release can be found
at [https://github.com/mrash/afl-cov/releases](https://github.com/mrash/afl-cov/releases)

## Contact
All feature requests and bug fixes are managed through github issues tracking.
However, you can email me (michael.rash_AT_gmail.com), or reach me through
Twitter ([@michaelrash](https://twitter.com/michaelrash)).
