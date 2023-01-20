---
doctest: -WINDOWS
---

# R script support

Guild provides support for R script via the `r_script` plugin.

## Setup R and the Guild AI package

Verify that R is available.

    >>> run("which Rscript")
    ???
    <exit 0>

Ensure that the Guild AI R package is installed.

    >>> quiet("mkdir -p /tmp/guild-uat/r-libs")
    >>> quiet("""test -d /tmp/guild-uat/r-libs/guildai || (
    ...     R_LIBS=/tmp/guild-uat/r-libs Rscript \
    ...     -e 'install.packages("remotes")' \
    ...     -e 'remotes::install_github("t-kalinowski/guildai-r")'
    ... )
    ... """)

Helper to run commands with `R_LIBS` set (needed for Rscript commands
and guild commands that need access to the Guild AI R package).

    >>> _run = run
    >>> def run(cmd):
    ...     _run(f"R_LIBS=/tmp/guild-uat/r-libs {cmd}")

Verify that the Guild AI package is available.

    >>> run("Rscript -e 'library(guildai)'")
    <exit 0>

Check R script info with `guild check`.

    >>> run("guild check --r-script --offline")
    guild_version:             ...
    rscript_version:           4.1...
    ...
    <exit 0>

## Simple hello world

The `r-plugin` example provides various samples of R based operations.

    >>> cd(example("r-plugin"))

Confirm the behavior in R directly.

    >>> run("Rscript hello.R")
    hello

    >>> run("R -q -f hello.R")
    > msg = 'hello'
    > cat(msg, '\n')
    hello
    >

Get Guild help for the script.

    >>> run("guild run hello.R --help-op")
    Usage: guild run [OPTIONS] hello.R [FLAG]...
    <BLANKLINE>
    Use 'guild run --help' for a list of options.
    <BLANKLINE>
    Flags:
      msg  (default is hello)

Run the script with Guild.

    >>> run("guild run hello.R -y")
    > msg = 'hello'
    > cat(msg, '\n')
    hello
    >
    >

Use a different message.

    >>> run("guild run hello.R msg=hi -y")
    > msg = "hi"
    > cat(msg, '\n')
    hi
    >
    >

Show our runs.

    >>> run("guild runs -s -n2")
    [1]  hello.R  completed  msg=hi
    [2]  hello.R  completed  msg=hello

Show runs using an operation filter.

    >>> run("guild runs -s -n2 -Fo hello.R")
    [1]  hello.R  completed  msg=hi
    [2]  hello.R  completed  msg=hello

Unlike the Python support for globals flag assignments, the R plugin
modifies the source code.

    >>> run("guild cat -p hello.R")
    msg = "hi"
    cat(msg, '\n')

R based programs should not generate a `pip_freeze` attribute.

    >>> run("guild select --attr pip_freeze")
    guild: no such run attribute 'pip_freeze'
    <exit 1>

## Run internal tests for Guild AI R package

TODO

---------------------------

R things to test:

- support of `config:<filename>` and `<filename>` - yaml and R files
- test unparseable R script
- test flag named 'y' (need to quote when we write attrs)
- test for loaded packages attr
- test assigments of every flag type (confirm that ints can be
  assigned to ints, etc.) - look for issues with ints and float vals

Other things to test:

- Yaml always written with newline

---------------------------

## Op data for R script

The R plugin examines R scripts to infer operation settings. We use
the `op_data_for_script` to illustrate.

    >> from guild.plugins.r_script import op_data_for_script

Helper function to print op data for a script along with any warnings
generated by the plugin:

    >> def op_data(script):
    ...     with LogCapture() as log:
    ...         data = op_data_for_script(script)
    ...     if log:
    ...         log.print_all()
    ...     pprint(data)

Empty R script:

    >> op_data("empty.R")
    WARNING: Error in if (is_anno[1]) { : missing value where TRUE/FALSE needed
    Calls: <Anonymous> -> print.yaml -> <Anonymous> -> r_script_guild_data
    Execution halted
    {}

Simple script:

    >> op_data("simple.R")
    {'exec': '...Rscript -e guildai:::\'do_guild_run("simple.R", '
             'flags_dest = "globals", echo = TRUE)\' ${flag_args}',
     'flags': {'layers': {'default': 32, 'type': 'int'},
               'name': {'default': 'foo', 'type': 'string'},
               'noise': {'default': 3.0, 'type': 'float'},
               'skip_connections': {'default': True, 'type': 'bool'}},
     'flags-dest': 'globals',
     'name': 'simple.R',
     'sourcecode': {'dest': '.'}}

## Run R scripts as Guild operations

    >> run("guild run simple.R -y")
    WARNING: unknown flag type 'bool' for skip_connections - cannot coerce
    WARNING: unknown option '--layers'
    <BLANKLINE>
    > noise = 3
    > layers <- 32L
    > layers = 4
    > name = "foo"
    > skip_connections = TRUE
    <exit 0>

## Notes

- Support empty file (edge case but we want this for test narrative)

- Flag type 'bool' -> 'boolean'

- If the Guild 'data' decoding is in a separate package, we might want
  to move that logic to the Python side even if it's implemented in R
  (interface via system process). This logic is central to what
  plugins provide and the dependency on another
  repo/build/deploy/install process and the versioning issues is
  probably something we want to avoid. If there is code shared across
  the two repos that could foil this thinking but we should look at
  some other options if we can thinking of any.

- The 'sourcecode' attr in simple.R look good - I wonder if the
  upstream warnings are somehow foiling Guild's correct copying of
  source code. They shouldn't - and that'd probably be a bug in
  Guild. But the config looks correct.