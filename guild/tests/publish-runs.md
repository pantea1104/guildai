# Publishing Runs

The `runs publish` commands is used to publish runs. A published run
is the public interface to a run. It consists of:

- Run info as formatted files
- Template generated report
- Run files
- Run source code

We'll use the `publish` sample project:

    >>> project = Project(sample("projects/publish"))

First, let's generate a run to publish:

    >>> project.run("op")
    x: 3
    y: 1

Here are our runs:

    >>> project.print_runs(flags=True, status=True)
    op  a=1 b=2  completed

We'll reference the run by its ID:

    >>> run_id = project.list_runs()[0].id

## Customized templates

Projects can define custom templates used in publishing. Templates are
used to generate reports for a published run. Guild supports a
flexible scheme that lets template authors selectively reuse and
override default template parts.

The tests below illustrate techniques that can be applied by custom
templates.

### Extending default and redefining blocks (template `t1`)

Template `t1` extends `publish-default/README.md` and redefines each
block in the default template.

    >>> publish_dest = mkdtemp()

    >>> project.publish(template="t1", dest=publish_dest)
    Publishing [...] op... using ...
    Refreshing runs index
    Published runs using ...

The files that are generated by the publish commant include the
template README.md, the run data files, and operation source code
(note that operation source code is defined by the operation as just
`op.py`).

    >>> find(path(publish_dest, run_id))
    ['README.md',
     'flags.yml',
     'output.txt',
     'run.yml',
     'runfiles.csv',
     'scalars.csv',
     'sourcecode.csv',
     'sourcecode/op.py']

 Here's the template generated report:

    >>> cat(path(publish_dest, run_id, "README.md"))
    Header
    <BLANKLINE>
    Title
    <BLANKLINE>
    Summary
    <BLANKLINE>
    Contents
    <BLANKLINE>
    Flags
    <BLANKLINE>
    Scalars
    <BLANKLINE>
    Files
    <BLANKLINE>
    Source Code
    <BLANKLINE>
    Output
    <BLANKLINE>
    Footer

### New template with include (template `t2`)

Template `t2` replaces the default entirely but includes the default
flags template `published-default/_flags.md`.

    >>> publish_dest = mkdtemp()

    >>> project.publish(template="t2", dest=publish_dest)
    Publishing [...] op... using ...
    Refreshing runs index
    Published runs using ...

    >>> dir(path(publish_dest, run_id))
    ['README.md',
     'flags.yml',
     'output.txt',
     'run.yml',
     'runfiles.csv',
     'scalars.csv',
     'some_other_file.md',
     'sourcecode',
     'sourcecode.csv']

    >>> cat(path(publish_dest, run_id, "README.md"))
    # Totally new report
    <BLANKLINE>
    Run: ...
    <BLANKLINE>
    ## Flags (included)
    <BLANKLINE>
    | Name | Value |
    | ---- | ----- |
    | a | 1 |
    | b | 2 |
    <BLANKLINE>
    [flags.yml](flags.yml)

### Extend default with block def and redefined include (template `t3`)

Template `t3` extends default and defines header and footer blocks. It
also redefines the `_flags.md` include.

    >>> publish_dest = mkdtemp()

    >>> project.publish(template="t3", dest=publish_dest)
    Publishing [...] op... using ...
    Refreshing runs index
    Published runs using ...

    >>> dir(path(publish_dest, run_id))
    ['README.md',
     'flags.yml',
     'output.txt',
     'run.yml',
     'runfiles.csv',
     'scalars.csv',
     'sourcecode',
     'sourcecode.csv']

    >>> cat(path(publish_dest, run_id, "README.md")) # doctest: +REPORT_UDIFF
    <BLANKLINE>
    Ze header
    <BLANKLINE>
    # op
    <BLANKLINE>
    | ID                   | Operation           | Started                  | Time                | Status           | Label                |
    | --                   | ---------           | ---------                | ----                | ------           | -----                |
    | ... | op | ... UTC | ... | completed | &nbsp; |
    <BLANKLINE>
    [run.yml](run.yml)
    <BLANKLINE>
    ## Contents
    <BLANKLINE>
    - [Flags](#flags)
    - [Scalars](#scalars)
    - [Run Files](#run-files)
    - [Source Code](#source-code)
    - [Output](#output)
    <BLANKLINE>
    ## Flags
    <BLANKLINE>
    Redef of flags
    <BLANKLINE>
    ## Scalars
    <BLANKLINE>
    | Key | Step | Value |
    | --- | ---- | ----- |
     | x | 0 | 3.0 |
     | y | 0 | 1.0 |
    <BLANKLINE>
    [scalars.csv](scalars.csv)
    <BLANKLINE>
    ## Run Files
    <BLANKLINE>
    | Path | Type | Size | Modified | MD5 |
    | ---- | ---- | ---- | -------- | --- |
    | generated-1.txt | file | 5 | ... UTC | 8c8432c5523c8507a5ec3b1ae3ab364f |
    | generated-2.txt | file | 9 | ... UTC | 7cbef5232b2e916eccc41d756d05035f |
    <BLANKLINE>
    [runfiles.csv](runfiles.csv)
    <BLANKLINE>
    ## Source Code
    <BLANKLINE>
    | Path | Size | Modified | MD5 |
    | ---- | ---- | -------- | --- |
    | [op.py](sourcecode/op.py) | ... | ... UTC | ... |
    <BLANKLINE>
    [sourcecode.csv](sourcecode.csv)
    <BLANKLINE>
    ## Output
    <BLANKLINE>
    ```
    x: 3
    y: 1
    ```
    <BLANKLINE>
    [output.txt](output.txt)
    <BLANKLINE>
    Ze footer

### Run files

Template `just-files` includes only the default run file template part
`publish-default/_runfiles.md`. We'll use this template to show how
run file lists are rendered by the default template.

Let's generate a run that includes a source link as well as generates
a new file (see `op3` operation in sample project):

    >>> project.run("op3")
    Resolving file:src.txt dependency
    x: 3
    y: 1

    >>> run_id = project.list_runs()[0].id

Publish using a `just-files` template (only prints the files table):

    >>> publish_dest = mkdtemp()
    >>> project.publish(["1"], template="just-files", dest=publish_dest)
    Publishing [...] op3... using ...
    Refreshing runs index
    Published runs using ...

Here are the published files:

    >>> find(path(publish_dest, run_id))
    ['README.md',
     'flags.yml',
     'output.txt',
     'run.yml',
     'runfiles.csv',
     'scalars.csv',
     'sourcecode.csv',
     'sourcecode/op.py']

And the generated report:

    >>> cat(path(publish_dest, run_id, "README.md"))
    | Path | Type | Size | Modified | MD5 |
    | ---- | ---- | ---- | -------- | --- |
    | generated-1.txt | file | 5 | ... UTC | 8c8432c5523c8507a5ec3b1ae3ab364f |
    | generated-2.txt | file | 9 | ... UTC | 7cbef5232b2e916eccc41d756d05035f |
    | link.txt | file link | 6 | ... UTC | 09f7e02f1290be211da707a266f153b3 |
    <BLANKLINE>
    [runfiles.csv](runfiles.csv)

Note that the file type for `link.txt` is a file link. Note also that
neither of the run files contain hyperlinks to their sources. That's
because the published run doesn't contain either run file.

#### Include files

Let's publish again using the `files` flag, which tells Guild to
publish files associated with the run:

    >>> project.publish(["1"],
    ...   template="just-files",
    ...   dest=publish_dest,
    ...   files=True)
    Publishing [...] op3... using ...
    Refreshing runs index
    Published runs using ...

This time the published run contains `generated-1.txt`:

    >>> find(path(publish_dest, run_id))
    ['README.md',
     'flags.yml',
     'output.txt',
     'run.yml',
     'runfiles.csv',
     'runfiles/generated-1.txt',
     'scalars.csv',
     'sourcecode.csv',
     'sourcecode/op.py']

And the report contains the applicable hyperlink:

    >>> cat(path(publish_dest, run_id, "README.md"))
    | Path | Type | Size | Modified | MD5 |
    | ---- | ---- | ---- | -------- | --- |
    | [generated-1.txt](runfiles/generated-1.txt) | file | 5 | ... UTC | 8c8432c5523c8507a5ec3b1ae3ab364f |
    | generated-2.txt | file | 9 | ... UTC | 7cbef5232b2e916eccc41d756d05035f |
    | link.txt | file link | 6 | ... UTC | 09f7e02f1290be211da707a266f153b3 |
    <BLANKLINE>
    [runfiles.csv](runfiles.csv)

Note that the published run does *not* contain `generated-2.txt`. This
is because `op3` specifies that `*.txt` be excluded from a published
files. Project authors can use this facility to control which files
are published and which are not.

#### Include all files

We can force all files to be published, including those that would
otherwise be excluded by an operation, by publishing with the
`all_files` flag:

    >>> project.publish(["1"],
    ...   template="just-files",
    ...   dest=publish_dest,
    ...   all_files=True)
    Publishing [...] op3... using ...
    Refreshing runs index
    Published runs using ...

The published run now contains both generated files:

    >>> find(path(publish_dest, run_id))
    ['README.md',
     'flags.yml',
     'output.txt',
     'run.yml',
     'runfiles.csv',
     'runfiles/generated-1.txt',
     'runfiles/generated-2.txt',
     'scalars.csv',
     'sourcecode.csv',
     'sourcecode/op.py']

 And the report contains the applicable hyperlinks:

    >>> cat(path(publish_dest, run_id, "README.md"))
    | Path | Type | Size | Modified | MD5 |
    | ---- | ---- | ---- | -------- | --- |
    | [generated-1.txt](runfiles/generated-1.txt) | file | 5 | ... UTC | 8c8432c5523c8507a5ec3b1ae3ab364f |
    | [generated-2.txt](runfiles/generated-2.txt) | file | 9 | ... UTC | 7cbef5232b2e916eccc41d756d05035f |
    | link.txt | file link | 6 | ... UTC | 09f7e02f1290be211da707a266f153b3 |
    <BLANKLINE>
    [runfiles.csv](runfiles.csv)

Note that `link.txt` is still not included in the published run
files. This is because Guild does not publish links by default. To
publish links, we must included the `include_links` flag.

#### Include links

Publish including links:

    >>> project.publish(["1"],
    ...   template="just-files",
    ...   dest=publish_dest,
    ...   include_links=True)
    Publishing [...] op3... using ...
    Refreshing runs index
    Published runs using ...

The files now contain `link.txt` as well as `generated-1.txt`:

    >>> find(path(publish_dest, run_id))
    ['README.md',
     'flags.yml',
     'output.txt',
     'run.yml',
     'runfiles.csv',
     'runfiles/generated-1.txt',
     'runfiles/link.txt',
     'scalars.csv',
     'sourcecode.csv',
     'sourcecode/op.py']

And the generated report contains the applicable links:

    >>> cat(path(publish_dest, run_id, "README.md"))
    | Path | Type | Size | Modified | MD5 |
    | ---- | ---- | ---- | -------- | --- |
    | [generated-1.txt](runfiles/generated-1.txt) | file | 5 | ... UTC | 8c8432c5523c8507a5ec3b1ae3ab364f |
    | generated-2.txt | file | 9 | ... UTC | 7cbef5232b2e916eccc41d756d05035f |
    | [link.txt](runfiles/link.txt) | file link | 6 | ... UTC | 09f7e02f1290be211da707a266f153b3 |
    <BLANKLINE>
    [runfiles.csv](runfiles.csv)

The use of `include_links` implies `files` flag. We can force all
files, including links, to be published by specifying both
`include_link` and `all_files` flags:

#### All files and links

Publish all files and links:

    >>> project.publish(["1"],
    ...   template="just-files",
    ...   dest=publish_dest,
    ...   all_files=True,
    ...   include_links=True)
    Publishing [...] op3... using ...
    Refreshing runs index
    Published runs using ...

Now all files are published:

    >>> find(path(publish_dest, run_id))
    ['README.md',
     'flags.yml',
     'output.txt',
     'run.yml',
     'runfiles.csv',
     'runfiles/generated-1.txt',
     'runfiles/generated-2.txt',
     'runfiles/link.txt',
     'scalars.csv',
     'sourcecode.csv',
     'sourcecode/op.py']

And each file has a hyperlink in the generated report:

    >>> cat(path(publish_dest, run_id, "README.md"))
    | Path | Type | Size | Modified | MD5 |
    | ---- | ---- | ---- | -------- | --- |
    | [generated-1.txt](runfiles/generated-1.txt) | file | 5 | ... UTC | 8c8432c5523c8507a5ec3b1ae3ab364f |
    | [generated-2.txt](runfiles/generated-2.txt) | file | 9 | ... UTC | 7cbef5232b2e916eccc41d756d05035f |
    | [link.txt](runfiles/link.txt) | file link | 6 | ... UTC | 09f7e02f1290be211da707a266f153b3 |
    <BLANKLINE>
    [runfiles.csv](runfiles.csv)

#### Missing link targets

NOTE: This test modifies the run directory for `run_id`.

If the link target is missing (e.g. it was deleted), the table shows
that it's missing.

To illustrate, we'll modify the run link to reference a non-existing
file:

    >>> link_path = path(project.guild_home, "runs", run_id, "link.txt")
    >>> os.remove(link_path)
    >>> os.symlink(path(project.cwd, "not-existing"), link_path)

Let's publish:

    >>> project.publish(["1"], template="just-files", dest=publish_dest)
    Publishing [...] op3... using ...
    Refreshing runs index
    Published runs using ...

The generated report contains the link reference, but does not contain
target file details:

    >>> cat(path(publish_dest, run_id, "README.md"))
    | Path | Type | Size | Modified | MD5 |
    | ---- | ---- | ---- | -------- | --- |
    | generated-1.txt | file | 5 | ... UTC | 8c8432c5523c8507a5ec3b1ae3ab364f |
    | generated-2.txt | file | 9 | ... UTC | 7cbef5232b2e916eccc41d756d05035f |
    | link.txt | link |  |  |  |
    <BLANKLINE>
    [runfiles.csv](runfiles.csv)