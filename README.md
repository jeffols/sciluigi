![SciLuigi Logo](http://i.imgur.com/2aMT04J.png)

* ***Watch a 10 minute screencast going through the basics of using SciLuigi [here](https://www.youtube.com/watch?v=gkKUWskRbjw)***
* ***See a poster describing the motivations behind SciLuigi [here](http://dx.doi.org/10.13140/RG.2.1.1143.6246)***

Scientific Luigi (SciLuigi for short) is a light-weight wrapper library around [Spotify](http://spotify.com)'s [Luigi](http://github.com/spotify/luigi)
workflow system that aims to make writing scientific workflows more fluent, flexible and
modular.

Luigi is a flexile and fun-to-use library. It has turned out though
that its default way of defining dependencies by hard coding them in each task's
requires() function is not optimal for some type of workflows common e.g. in bioinformatics where multiple inputs and outputs, complex dependencies,
and the need to quickly try different workflow connectivity in an explorative fashion is central to the way of working.

SciLuigi was designed to solve some of these problems, by providing the following
"features" over vanilla Luigi:

- Separation of dependency definitions from the tasks themselves,
  for improved modularity and composability.
- Inputs and outputs implemented as separate fields, a.k.a.
  "ports", to allow specifying dependencies between specific input
  and output-targets rather than just between tasks. This is again to let such
  details of the network definition reside outside the tasks.
- The fact that inputs and outputs are object fields, also allows auto-completion
  support to ease the network connection work (Works great e.g. with [jedi-vim](https://github.com/davidhalter/jedi-vim)).
- Inputs and outputs are connected with an intuitive "single-assignment syntax".
- "Good default" high-level logging of workflow tasks and execution times.
- Produces an easy to read audit-report with high level information per task.
- Integration with some HPC workload managers.
  (So far only [SLURM](http://slurm.schedmd.com/) though).

Because of Luigi's easy-to-use API these changes have been implemented
as a very thin layer on top of luigi's own API with no changes at all to the luigi
core, which means that you can continue leveraging the work already being
put into maintaining and further developing luigi by the team at Spotify and others.

## Workflow code quick demo

***For a brief 10 minute screencast going through the basics below, see [this link](https://www.youtube.com/watch?v=gkKUWskRbjw)***

Just to give a quick feel for how a workflow definition might look like in SciLuigi, check this code example
(implementation of tasks hidden here for brevity. See Usage section further below for more details):

```python
import sciluigi as sl

class MyWorkflow(sl.WorkflowTask):
    def workflow(self):
        # Initialize tasks:
        foowrt = self.new_task('foowriter', MyFooWriter)
        foorpl = self.new_task('fooreplacer', MyFooReplacer,
            replacement='bar')

        # Here we do the *magic*: Connecting outputs to inputs:
        foorpl.in_foo = foowrt.out_foo

        # Return the last task(s) in the workflow chain.
        return foorpl
```

That's it! And again, see the "usage" section just below for a more detailed description of getting to this!

## Prerequisites

- Python 2.7 - 3.4
- Luigi 1.3.x

## Install

1. Install SciLuigi, including its dependencies (luigi etc), through PyPI:

    ```bash
    pip install sciluigi
    ```

2. Now you can use the library by just importing it in your python script, like so:

    ```python
    import sciluigi
    ```

    Note that you can aliase it to a shorter name, for brevity, and to save keystrokes:

    ```python
    import sciluigi as sl
    ```

## Usage

Creating workflows in SciLuigi differs slightly from how it is done in vanilla Luigi.
Very briefly, it is done in these main steps:

1. Create a workflow tasks clas
2. Create task classes
3. Add the workflow definition in the workflow class's `worklfow()` method.
4. Add a run method at the end of the script
5. Run the script

### Create a Workflow task

The first thing to do when creating a workflow, is to define a workflow task.

You do this by:

1. Creating a subclass of `sciluigi.WorkflowTask`
2. Implementing the `workflow()` method.

#### Example:

```python
import sciluigi

class MyWorkflow(sciluigi.WorkflowTask):
    def workflow(self):
        pass # TODO: Implement workflow here later!
```

### Create tasks

Then, you need to define some tasks that can be done in this workflow.

This is done by:

1. Creating a subclass of `sciluigi.Task` (or `sciluigi.SlurmTask` if you want Slurm support)
2. Adding fields named `in_<yournamehere>` for each input, in the new task class
3. Define methods named `out_<yournamehere>()` for each output, that return `sciluigi.TargetInfo` objects. (sciluigi.TargetInfo is initialized with a reference to the task object itself - typically `self` - and a path name, where upstream tasks paths can be used).
4. Define luigi parameters to the task.
5. Implement the `run()` method of the task.

#### Example:

Let's define a simple task that just writes "foo" to a file named `foo.txt`:

```python
class MyFooWriter(sciluigi.Task):
    # We have no inputs here
    # Define outputs:
    def out_foo(self):
        return sciluigi.TargetInfo(self, 'foo.txt')
    def run(self):
        with self.out_foo().open('w') as foofile:
            foofile.write('foo\n')
```

Then, let's create a task taht replaces "foo" with "bar":

```python
class MyFooReplacer(sciluigi.Task):
    replacement = sciluigi.Parameter() # Here, we take as a parameter
                                  # what to replace foo with.
    # Here we have one input, a "foo file":
    in_foo = None
    # ... and an output, a "bar file":
    def out_replaced(self):
        # As the path to the returned target(info), we
        # use the path of the foo file:
        return sciluigi.TargetInfo(self, self.in_foo().path + '.bar.txt')
    def run(self):
        with self.in_foo().open() as in_f:
            with self.out_replaced().open('w') as out_f:
                # Here we see that we use the parameter self.replacement:
                out_f.write(in_f.read().replace('foo', self.replacement))
```

The last lines, we could have instead written using the command-line `sed` utility, available in linux, by calling it on the commandline, with the built-in `ex()` method:

```python
    def run(self):
        # Here, we use the in-built self.ex() method, to execute commands:
        self.ex("sed 's/foo/{repl}' {in} > {out}".format(
            repl=self.replacement,
            in=self.in_foo().path,
            out=self.out_bar().path))
```

### Write the workflow definition

Now, we can use these two tasks we created, to create a simple workflow, in our workflow class, that we also created above.

We do this by:

1. Instantiating the tasks, using the `self.new_task(<unique_taskname>, <task_class>, *args, **kwargs)` method, of the workflow task.
2. Connect the tasks together, by pointing the right `out_*` method to the right `in_*` field.
3. Returning the last task in the chain, from the workflow method.

#### Example:

```python
import sciluigi
class MyWorkflow(sciluigi.WorkflowTask):
    def workflow(self):
        foowriter = self.new_task('foowriter', MyFooWriter)
        fooreplacer = self.new_task('fooreplacer', MyFooReplacer,
            replacement='bar')

        # Here we do the *magic*: Connecting outputs to inputs:
        fooreplacer.in_foo = foowriter.out_foo

        # Return the last task(s) in the workflow chain.
        return fooreplacer
```

### Add a run method to the end of the script

Now, the only thing that remains, is adding a run method to the end of the script.

You can use luigi's own `luigi.run()`, or our own two methods:

1. `sciluigi.run()`
2. `sciluigi.run_local()`

The `run_local()` one, is handy if you don't want to run a central scheduler daemon, but just want to run the workflow as a script.

Both of the above take the same options as `luigi.run()`, so you can for example set the main class to use (our workflow task):

```
# End of script ....
if __name__ == '__main__':
    sciluigi.run_local(main_task_cls=MyWorkflow)
```

### Run the workflow

Now, you should be able to run the workflow as simple as:

```bash
python myworkflow.py
```

... provided of course, that the workflow is saved in a file named myworkflow.py.

### More Examples

See the [examples folder](https://github.com/samuell/sciluigi/tree/master/examples) for more detailed examples!

### More links, background info etc.

The basic idea behind SciLuigi, and a preceding solution to it, was
presented in workshop (e-Infra MPS 2015) talk:
- [Slides](http://www.slideshare.net/SamuelLampa/building-workflows-with-spotifys-luigi)
- [Video](https://www.youtube.com/watch?v=f26PqSXZdWM)

See also [this collection of links](http://bionics.it/posts/our-experiences-using-spotifys-luigi-for-bioinformatics-workflows), to more of our reported experiences
using Luigi, which lead up to the creation of SciLuigi.

Changelog
---------
- 0.9.3b4
  - Support for Python 3 (Thanks to @jeffcjohnson for contributing this!).
  - Bug fixes.

Contributors
----------------
- [Samuel Lampa](https://github.com/samuell)
- [Jeff C Johnson](https://github.com/jeffcjohnson)

Acknowledgements
----------------
This work is funded by:
- [Faculty grants of the dept. of Pharmaceutical Biosciences, Uppsala University](http://www.farmbio.uu.se)
- [Bioinformatics Infrastructure for Life Sciences, BILS](https://bils.se)

Many ideas and inspiration for the API is taken from:
- [John Paul Morrison's invention and works on Flow-Based Programming](jpaulmorrison.com/fbp)
