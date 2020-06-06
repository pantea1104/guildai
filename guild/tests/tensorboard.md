# TensorBoard

These are miscellaneous checks for TensorBoard support.

    >> from guild import tensorboard

## TensorBoard config info

We run `guild.tensorboard` as a main module to capture output. This
insulates us from spurious warnings that Google libraries are prone to
log directly to a tty rather than via the Python logging system.

    >>> import guild
    >>> import subprocess

    >>> env = dict(os.environ)
    >>> env["PYTHONPATH"] = guild.__pkgdir__
    >>> p = subprocess.Popen(
    ...     [sys.executable, "-m", "guild.tensorboard"],
    ...     env=env,
    ...     stdout=subprocess.PIPE,
    ...     stderr=subprocess.PIPE)
    >>> out, err = p.communicate()

    >>> p.returncode, err
    (0, ...)

Output generated by `guild.tensorboard` when run as main is JSON
containing TensorBoard info.

    >>> import json
    >>> data = json.loads(out.decode())

    >>> sorted(data)
    ['plugins', 'version']

TensorBoard version under Python 2:

    >>> data["version"]  # doctest: -PY3
    '2.1.0'

TensorBoard version under Python 3:

    >>> data["version"]  # doctest: -PY2
    '2.2.2'

Plugins under Python 2:

    >>> for p in data["plugins"]:
    ...     print(p)  # doctest: +REPORT_UDIFF -PY3
    tensorboard.plugins.core.core_plugin.CorePluginLoader
    tensorboard.plugins.scalar.scalars_plugin.ScalarsPlugin
    tensorboard.plugins.custom_scalar.custom_scalars_plugin.CustomScalarsPlugin
    tensorboard.plugins.image.images_plugin.ImagesPlugin
    tensorboard.plugins.audio.audio_plugin.AudioPlugin
    tensorboard.plugins.debugger.debugger_plugin_loader.DebuggerPluginLoader
    tensorboard.plugins.graph.graphs_plugin.GraphsPlugin
    tensorboard.plugins.distribution.distributions_plugin.DistributionsPlugin
    tensorboard.plugins.histogram.histograms_plugin.HistogramsPlugin
    tensorboard.plugins.text.text_plugin.TextPlugin
    tensorboard.plugins.pr_curve.pr_curves_plugin.PrCurvesPlugin
    tensorboard.plugins.profile.profile_plugin_loader.ProfilePluginLoader
    tensorboard.plugins.beholder.beholder_plugin_loader.BeholderPluginLoader
    tensorboard.plugins.interactive_inference.interactive_inference_plugin_loader.InteractiveInferencePluginLoader
    tensorboard.plugins.hparams.hparams_plugin.HParamsPlugin
    tensorboard.plugins.mesh.mesh_plugin.MeshPlugin
    tensorboard.plugins.projector.projector_plugin.ProjectorPlugin

Plugins under Python 3 (difference in underlying TB version):

    >>> for p in data["plugins"]:
    ...     print(p)  # doctest: +REPORT_UDIFF -PY2
    tensorboard.plugins.core.core_plugin.CorePluginLoader
    tensorboard.plugins.scalar.scalars_plugin.ScalarsPlugin
    tensorboard.plugins.custom_scalar.custom_scalars_plugin.CustomScalarsPlugin
    tensorboard.default.ExperimentalDebuggerV2Plugin
    tensorboard.plugins.image.images_plugin.ImagesPlugin
    tensorboard.plugins.audio.audio_plugin.AudioPlugin
    tensorboard.plugins.debugger.debugger_plugin_loader.DebuggerPluginLoader
    tensorboard.plugins.graph.graphs_plugin.GraphsPlugin
    tensorboard.plugins.distribution.distributions_plugin.DistributionsPlugin
    tensorboard.plugins.histogram.histograms_plugin.HistogramsPlugin
    tensorboard.plugins.text.text_plugin.TextPlugin
    tensorboard.plugins.pr_curve.pr_curves_plugin.PrCurvesPlugin
    tensorboard.plugins.profile_redirect.profile_redirect_plugin.ProfileRedirectPluginLoader
    tensorboard.plugins.beholder.beholder_plugin_loader.BeholderPluginLoader
    tensorboard.plugins.hparams.hparams_plugin.HParamsPlugin
    tensorboard.plugins.mesh.mesh_plugin.MeshPlugin
    tensorboard.plugins.projector.projector_plugin.ProjectorPlugin
    tensorboard_plugin_wit.wit_plugin_loader.WhatIfToolPluginLoader

## Runs Monitor

The class `guild.tensorboard.RunsMonitor` is responsible for running
in a background thread that monitors a list of runs and applies
changes as needed to a TB log directory.

    >>> from guild.tensorboard import RunsMonitor

We a log directory.

    >>> logdir = mkdtemp()

For our runs, we use the `tensorboard` project.

    >>> project = Project(sample("projects", "tensorboard"))

We can use the project `runs` method as the run callback for the
monitor.

We can also provide a function for naming the run directories the
monitor uses.

    >>> run_name = lambda run: "run-%s" % run.short_id

Our monitor:

    >>> monitor = RunsMonitor(logdir, project.list_runs, run_name_cb=run_name)

Run monitors are typically run in a polling thread. For our tests, we
use the `run_once` method to explicitly check each step of our tests.

Our log directory is initially empty.

    >>> find(logdir)
    <empty>

With no runs, there's nothing to setup.

    >>> with LogCapture(log_level=0) as log:
    ...     monitor.run_once()

    >>> log.print_all()
    DEBUG: [guild] Refreshing runs

    >>> find(logdir)
    <empty>

Let's run three sample operations.

    >>> project.run("op")
    m1: 1.123

    >>> project.run("op", flags={"a": 2.0, "c": "hola"})
    m1: 1.123

    >>> project.run("op", flags={"a": 3, "b": "two", "c": "bonjour"})
    m1: 1.123

    >>> project.print_runs(flags=True)
    op  a=3 b=two c=bonjour d=yes extra_metrics=0
    op  a=2.0 b=2 c=hola d=yes extra_metrics=0
    op  a=1.0 b=2 c=hello d=yes extra_metrics=0

The combination of these runs defines the hyperparameters (hparams)
and metrics setup by the runs monitor. The hparam TB plugin does not
modify the hparam settings when it reads subsequent experiment
summaries. Any runs the monitor handles after this first set must fit
within the hparam domain scheme to be displayed as expected in the
hparam tab.

We run three runs of specific values to establish the following hparam
scheme:

- Flag `a` consists of three numbers, so is created as an unbounded
  `RealInterval` by the monitor.

- Flag `b` is of mixed type and so will not have a domain. The hparam
  interface does not support filtering or log scale options for
  hparams without a domain.

- Flag `c` consists of three strings: hello, hola, and bonjour. These
  are defined using a `Discrete` domain, which supports filtering by
  checking and unchecking values.

- Flag `d` conists only of boolean values and will be created with a
  special `Discrete` domain consisting of only True and False
  options. Note that even though False does not appear in any of the
  runs, the monitor adds it as an option.

Let's run the monitor, capturing debug output.

    >>> with LogCapture(log_level=0) as log:
    ...     monitor.run_once()

The monitor generates the following logs:

    >> log.print_all()
    DEBUG: [guild] Refreshing runs
    DEBUG: [guild] hparam experiment:
           hparams=['a', 'c', 'b', 'd', 'extra_metrics', 'sourcecode']
           metrics=['m1', 'time']
    DEBUG: [guild] Creating link from '.../.guild/events.out.tfevents...' to '.../.guild/events.out.tfevents...'
    DEBUG: [guild] hparam experiment:
           hparams=['a', 'c', 'b', 'd', 'extra_metrics', 'sourcecode']
           metrics=['m1', 'time']
    DEBUG: [guild] Creating link from '.../.guild/events.out.tfevents...' to '.../.guild/events.out.tfevents...'
    DEBUG: [guild] hparam experiment:
           hparams=['a', 'c', 'b', 'd', 'extra_metrics', 'sourcecode']
           metrics=['m1', 'time']
    DEBUG: [guild] Creating link from '.../.guild/events.out.tfevents...' to '.../.guild/events.out.tfevents...'

Note the following from the log output:

- The experiment summaries are the same. Guild writes new summaries
  for each run using the combined information on hparms and metrics
  for all available runs.

- Each run events log is linked within the monitor log dir.

And the log dir:

    >>> find(logdir)
    run-.../.guild/events.out.tfevents.0000000000.hparams
    run-.../.guild/events.out.tfevents...
    run-.../.guild/events.out.tfevents.9999999999.time
    run-.../.guild/events.out.tfevents.0000000000.hparams
    run-.../.guild/events.out.tfevents...
    run-.../.guild/events.out.tfevents.9999999999.time
    run-.../.guild/events.out.tfevents.0000000000.hparams
    run-.../.guild/events.out.tfevents...
    run-.../.guild/events.out.tfevents.9999999999.time

For subsequent tests in this section we assert file count in logdir.

    >>> len(findl(logdir))
    9


Now the fun begins! Each new run that arrives may or may not fall
within the scheme inferred from the first set of runs. Runs that fall
within the scheme are logged without warning. Runs that do not
generate one or more warnings.

Let's generate a run that falls within the scheme, specifically:

- `a` is a number
- `b` can be anything (it was mixed to begin with)
- `c` is one of the three known string values
- `d` is boolean

    >>> project.run("op", flags={"a": 0, "b": False, "c": "hola", "d": False})
    m1: 1.123

With this new run, let's run the monitor.

    >>> with LogCapture(log_level=0) as log:
    ...     monitor.run_once()

    >>> log.print_all()
    DEBUG: [guild] Refreshing runs
    DEBUG: [guild] hparam experiment:
           hparams=['a', 'b', 'c', 'd', 'extra_metrics', 'sourcecode']
           metrics=['m1', 'time']
    DEBUG: [guild] Creating link ...

Note the same hparam experiment scheme.

Next we generate a run that falls outside the scheme.

- `a` is not a number
- `c` is not one of the three original values
- `d` is not a boolean

    >>> project.run("op", flags={"a": "0", "c": "bye", "d": 1})
    m1: 1.123

When we ask the monitor to process this run, it logs warnings about
each incompatibility.

    >>> with LogCapture(log_level=0, strip_ansi_format=True) as log:
    ...     monitor.run_once()

    >>> log.print_all()
    DEBUG: [guild] Refreshing runs
    WARNING: Runs found with hyperparameter values that cannot be displayed in the
             HPARAMS plugin: a='0', c=bye, d=1. Restart this command to view them.
    DEBUG: [guild] hparam experiment:
           hparams=['a', 'b', 'c', 'd', 'extra_metrics', 'sourcecode']
           metrics=['m1', 'time']
    DEBUG: [guild] Creating link ...

Next we generate a run with flags that weren't in the original set.

    >>> project.run("op", flags={"e": 123}, force_flags=True)
    m1: 1.123

In this case, the monitor warns of new hyperparameters.

    >>> with LogCapture(log_level=0, strip_ansi_format=True) as log:
    ...     monitor.run_once()

    >>> log.print_all()
    DEBUG: [guild] Refreshing runs
    WARNING: Runs found with new hyperparameters: e. These hyperparameters
             will NOT appear in the HPARAMS plugin. Restart this command to view them.
    WARNING: Runs found with hyperparameter values that cannot be displayed in the
             HPARAMS plugin: a='0', c=bye. Restart this command to view them.
    DEBUG: [guild] hparam experiment:
           hparams=['a', 'b', 'c', 'd', 'extra_metrics', 'sourcecode']
           metrics=['m1', 'time']
    DEBUG: [guild] Creating link ...

Note that the monitor warns of the full set of incompatible hparam
items - not just of the more recently added run. This is
less-than-ideal as it is preferable to include specific about each
run, but this approach is sufficient to call attention to the issue.

Finally, we generate a run that logs scalars that are not part of the
original set. This too causes the monitor to log a warning.

    >>> project.run("op", flags={"extra_metrics": 2})
    m1: 1.123
    m2: 1.123
    m3: 1.123

    >>> with LogCapture(log_level=0, strip_ansi_format=True) as log:
    ...     monitor.run_once()

    >>> log.print_all()
    DEBUG: [guild] Refreshing runs
    WARNING: Runs found with new hyperparameters: e. These hyperparameters
             will NOT appear in the HPARAMS plugin. Restart this command to view them.
    WARNING: Runs found with hyperparameter values that cannot be displayed in the
             HPARAMS plugin: a='0', c=bye. Restart this command to view them.
    WARNING: Runs found with new metrics: m2, m3. These runs will NOT appear in the
             HPARAMS plugin. Restart this command to view them.
    DEBUG: [guild] hparam experiment:
           hparams=['a', 'b', 'c', 'd', 'extra_metrics', 'sourcecode']
           metrics=['m1', 'time']
    DEBUG: [guild] Creating link ...