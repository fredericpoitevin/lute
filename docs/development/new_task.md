# Integrating a New `Task`

`Task`s can be broadly categorized into two types:

- "First-party" - where the analysis or executed code is maintained within this library.
- "Third-party" - where the analysis, code, or program is maintained elsewhere and is simply called by a wrapping `Task`.

Creating a new `Task` of either type generally involves the same steps, although for first-party `Task`s, the analysis code must of course also be written. Due to this difference, as well as additional considerations for parameter handling when dealing with "third-party" `Task`s, the "first-party" and "third-party" `Task` integration cases will be considered separately.

## Creating a "Third-party" `Task`

There are two required steps for third-party `Task` integration, and one additional step which is optional, and may not be applicable to all possible third-party `Task`s. Generally, `Task` integration requires:

1. Defining a `TaskParameters` (pydantic) model which fully parameterizes the `Task`. This involves specifying a path to a binary, and all the required command-line arguments to run the binary.
2. Creating a **managed `Task`** by specifying an `Executor` for the new third-party `Task`. At this stage, any additional environment variables can be added which are required for the execution environment.
3. **(Optional/Maybe applicable)** Create a template for a third-party configuration file. If the new `Task` has its own configuration file, specifying a template will allow that file to be parameterized from the singular LUTE yaml configuration file. A couple of minor additions to the `pydantic` model specified in 1. are required to support template usage.

Each of these stages will be discussed in detail below. The vast majority of the work is completed in step 1.

### Specifying a `TaskParameters` Model for your `Task`

A brief overview of parameters objects will be provided below. The following information goes into detail only about specifics related to LUTE configuration. An in depth description of pydantic is beyond the scope of this tutorial; please refer to the [official documentation](https://docs.pydantic.dev/1.10/) for more information. Please note that due to environment constraints pydantic is currently pinned to version 1.10! Make sure to read the appropriate documentation for this version as many things are different compared to the newer releases. At the end this document there will be an example highlighting some supported behaviour as well as a FAQ to address some common integration considerations.

**`Task`s and `TaskParameter`s**

All `Task`s have a corresponding `TaskParameters` object. These objects are linked **exclusively** by a named relationship. For a `Task` named `MyThirdPartyTask`, the parameters object **must** be named `MyThirdPartyTaskParameters`. For third-party `Task`s there are a number of additional requirements:

- The model must inherit from a base class called `ThirdPartyParameters`.
- The model must have one field specified called `executable`. The presence of this field indicates that the `Task` is a third-party `Task` and the specified executable must be called. This allows all third-party `Task`s to be defined exclusively by their parameters model. A single `ThirdPartyTask` class handles execution of **all** third-party `Task`s.

All models are stored in `lute/io/models`. For any given `Task`, a new model can be added to an existing module contained in this directory or to a new module. If creating a new module, make sure to add an import statement to `lute.io.models.__init__`.

**Defining `TaskParameter`s**

When specifying parameters the default behaviour is to provide a one-to-one correspondance between the Python attribute specified in the parameter model, and the parameter specified on the command-line. Single-letter attributes are assumed to be passed using `-`, e.g. `n` will be passed as `-n` when the executable is launched. Longer attributes are passed using `--`, e.g. by default a model attribute named `my_arg` will be passed on the command-line as `--my_arg`. Positional arguments are specified using `p_argX` where `X` is a number. All parameters are passed in the order that they are specified in the model.

However, because the number of possible command-line combinations is large, relying on the default behaviour above is **NOT recommended**. It is provided solely as a fallback. Instead, there are a number of configuration knobs which can be tuned to achieve the desired behaviour. The two main mechanisms for controlling behaviour are specification of model-wide configuration under the `Config` class within the model's definition, and parameter-by-parameter configuration using field attributes. For the latter, we define all parameters as `Field` objects. This allows parameters to have their own attributes, which are parsed by LUTE's task-layer. Given this, the preferred starting template for a `TaskParameters` model is the following - we assume we are integrating a new `Task` called `RunTask`:

```py

from pydantic import Field, validator
# Also include any pydantic type specifications - Pydantic has many custom
# validation types already, e.g. types for constrained numberic values, URL handling, etc.

from .base import ThirdPartyParameters

# Change class name as necessary
class RunTaskParameters(ThirdPartyParameters):
    """Parameters for RunTask..."""

    class Config(ThirdPartyParameters.Config): # MUST be exactly as written here.
        ...
        # Model-wide configuration will go here

    executable: str = Field("/path/to/executable", description="...")
    ...
    # Additional params.
    # param1: param1Type = Field("default", description="", ...)
```


**Config settings and options**
Under the class definition for `Config` in the model, we can modify global options for all the parameters. In addition, there are a number of configuration options related to specifying what the outputs/results from the associated `Task` are, and a number of options to modify runtime behaviour. Currently, the available configuration options are:

| **Config Parameter** | **Meaning**                                                                                                  | **Default Value**     | **ThirdPartyTask-specific?**             |
|:--------------------:|:------------------------------------------------------------------------------------------------------------:|:---------------------:|:----------------------------------------:|
| `run_directory`      | If provided, can be used to specify the directory from which a `Task` is run.                                | `None` (not provided) | **NO**                                   |
| `set_result`         | `bool`. If `True` search the model definition for a parameter that indicates what the result is.             | `False`               | **NO**                                   |
| `result_from_params` | If `set_result` is `True` can define a result using this option and a validator. See also `is_result` below. | `None` (not provided) | **NO**                                   |
| `short_flags_use_eq` | Use equals sign instead of space for arguments of `-` parameters.                                            | `False`               | **YES** - Only affects `ThirdPartyTask`s |
| `long_flags_use_eq`  | Use equals sign instead of space for arguments of `-` parameters.                                            | `False`               | **YES** - Only affects `ThirdPartyTask`s |
|                      |                                                                                                              |                       |                                          |

These configuration options modify how the parameter models are parsed and passed along on the command-line, as well as what we consider results and where a `Task` can run. The default behaviour is that parameters are assumed to be passed as `-p arg` and `--param arg`, the `Task` will be run in the current working directory (or scratch if submitted with the ARP), and we have no information about `Task` results . Setting the above options can modify this behaviour.

- By setting `short_flags_use_eq` and/or `long_flags_use_eq` to `True` parameters are instead passed as `-p=arg` and `--param=arg`.
- By setting `run_directory` to a valid path, we can force a `Task` to be run in a specific directory. By default the `Task` will be run from the directory you submit the job in, or from your scratch folder (`/sdf/scratch/...`) if you submit from the eLog. Some `ThirdPartyTask`s rely on searching the correct working directory in order run properly.
- By setting `set_result` to `True` we indicate that the `TaskParameters` model will provide information on what the `TaskResult` is. This setting must be used with one of two options, either the `result_from_params` `Config` option, described below, or the **Field** attribute `is_result` described in the next sub-section (**Field Attributes**).
- `result_from_params` is a Config option that can be used when `set_result==True`. In conjunction with a **validator** (described a sections down) we can use this option to specify a result from all the information contained in the model. E.g. if you have a `Task` that has parameters for an `output_directory` and a `output_filename`, you can set `result_from_params==f"{output_directory}/{output_filename}"`.


**Field attributes**
In addition to the global configuration options there are a couple of ways to specify individual parameters. The following `Field` attributes are used when parsing the model:

| **Field Attribute** | **Meaning**                                                                                            | **Default Value** | **Example**                                       |
|:-------------------:|:------------------------------------------------------------------------------------------------------:|:-----------------:|:-------------------------------------------------:|
| `flag_type`         | Specify the type of flag for passing this argument. One of `"-"`, `"--"`, or `""`                      | N/A               | `p_arg1 = Field(..., flag_type="")`               |
| `rename_param`      | Change the name of the parameter as passed on the command-line.                                        | N/A               | `my_arg = Field(..., rename_param="my-arg")`      |
| `description`       | Documentation of the parameter's usage or purpose.                                                     | N/A               | `arg = Field(..., description="Argument for...")` |
| `is_result`         | `bool`. If the `set_result` `Config` option is `True`, we can set this to `True` to indicate a result. | N/A               | `output_result = Field(..., is_result=true)`      |
|                     |                                                                                                        |                   |                                                   |

The `flag_type` attribute allows us to specify whether the parameter corresponds to a positional (`""`) command line argument, requires a single hyphen (`"-"`), or a double hyphen (`"--"`). By default, the parameter name is passed as-is on the command-line. However, command-line arguments can have characters which would not be valid in Python variable names. In particular, hyphens are frequently used. To handle this case, the `rename_param` attribute can be used to specify an alternative spelling of the parameter when it is passed on the command-line. This also allows for using more descriptive variable names internally than those used on the command-line. A `description` can also be provided for each Field to document the usage and purpose of that particular parameter.

As an example, we can again consider defining a model for a `RunTask` `Task`. Consider an executable which would normally be called from the command-line as follows:
```bash
/sdf/group/lcls/ds/tools/runtask -n <nthreads> --method=<algorithm> -p <algo_param> [--debug]
```

A model specification for this `Task` may look like:
```py
class RunTaskParameters(ThirdPartyParameters):
    """Parameters for the runtask binary."""

    class Config(ThirdPartyParameters.Config):
        long_flags_use_eq: bool = True  # For the --method parameter

    # Prefer using full/absolute paths where possible.
    # No flag_type needed for this field
    executable: str = Field(
        "/sdf/group/lcls/ds/tools/runtask", description="Runtask Binary v1.0"
    )

    # We can provide a more descriptive name for -n
    # Let's assume it's a number of threads, or processes, etc.
    num_threads: int = Field(
        1, description="Number of concurrent threads.", flag_type="-", rename_param="n"
    )

    # In this case we will use the Python variable name directly when passing
    # the parameter on the command-line
    method: str = Field("algo1", description="Algorithm to use.", flag_type="--")

    # For an actual parameter we would probably have a better name. Lets assume
    # This parameter (-p) modifies the behaviour of the method above.
    method_param1: int = Field(
        3, description="Modify method performance.", flag_type="-", rename_param="p"
    )

    # Boolean flags are only passed when True! `--debug` is an optional parameter
    # which is not followed by any arguments.
    debug: bool = Field(
        False, description="Whether to run in debug mode.", flag_type="--"
    )
```

The `is_result` attribute allows us to specify whether the corresponding Field points to the output/result of the associated `Task`. Consider a `Task`, `RunTask2` which writes its output to a single file which is passed as a parameter.

```py
class RunTask2Parameters(ThirdPartyParameters):
    """Parameters for the runtask2 binary."""

    class Config(ThirdPartyParameters.Config):
        set_result: bool = True                     # This must be set here!
        # result_from_params: Optional[str] = None  # We can use this for more complex result setups (see below). Ignore for now.

    # Prefer using full/absolute paths where possible.
    # No flag_type needed for this field
    executable: str = Field(
        "/sdf/group/lcls/ds/tools/runtask2", description="Runtask Binary v2.0"
    )

    # Lets assume we take one input and write one output file
    # We will not provide a default value, so this parameter MUST be provided
    input: str = Field(
        description="Path to input file.", flag_type="--"
    )

    # We will also not provide a default for the output
    # BUT, we will specify that whatever is provided is the result
    output: str = Field(
        description="Path to write output to.",
        flag_type="-",
        rename_param="o",
        is_result=True,   # This means this parameter points to the result!
    )
```

**Additional Comments**

1. Model parameters of type `bool` are not passed with an argument and are only passed when `True`. This is a common use-case for boolean flags which enable things like test or debug modes, verbosity or reporting features. E.g. `--debug`, `--test`, `--verbose`, etc.
  - If you need to pass the literal words `"True"` or `"False"`, use a parameter of type `str`.
2. You can use `pydantic` types to constrain parameters beyond the basic Python types. E.g. `conint` can be used to define lower and upper bounds for an integer. There are also types for common categories, positive/negative numbers, paths, URLs, IP addresses, etc.
  - Even more custom behaviour can be achieved with `validator`s (see below).
3. All `TaskParameters` objects and its subclasses have access to a `lute_config` parameter, which is of type `lute.io.models.base.AnalysisHeader`. This special parameter is ignored when constructing the call for a binary task, but it provides access to shared/common parameters between tasks. For example, the following parameters are available through the `lute_config` object, and may be of use when constructing validators. All fields can be accessed with `.` notation. E.g. `lute_config.experiment`.
  - `title`: A user provided title/description of the analysis.
  - `experiment`: The current experiment name
  - `run`: The current acquisition run number
  - `date`: The date of the experiment or the analysis.
  - `lute_version`: The version of the software you are running.
  - `task_timeout`: How long a `Task` can run before it is killed.
  - `work_dir`: The main working directory for LUTE. Files and the database are created relative to this directory. This is separate from the `run_directory` config option. LUTE will write files to the work directory by default; however, the `Task` itself is run from `run_directory` if it is specified.

**Validators**
Pydantic uses `validators` to determine whether a value for a specific field is appropriate. There are default validators for all the standard library types and the types specified within the pydantic package; however, it is straightforward to define custom ones as well. In the template code-snippet above we imported the `validator` decorator. To create our own validator we define a method (with any name) with the following prototype, and decorate it with the `validator` decorator:
```py
@validator("name_of_field_to_decorate")
def my_custom_validator(cls, field: Any, values: Dict[str, Any]) -> Any: ...
```
In this snippet, the `field` variable corresponds to the value for the specific field we want to validate. `values` is a dictionary of fields and their values which have been parsed prior to the current field. This means you can validate the value of a parameter based on the values provided for other parameters. Since pydantic always validates the fields in the order they are defined in the model, fields dependent on other fields should come later in the definition.

For example, consider the `method_param1` field defined above for `RunTask`. We can provide a custom validator which changes the default value for this field depending on what type of algorithm is specified for the `--method` option. We will also constrain the options for `method` to two specific strings.

```py
from pydantic import Field, validator, ValidationError, root_validator
class RunTaskParameters(ThirdPartyParameters):
    """Parameters for the runtask binary."""

    # [...]

    # In this case we will use the Python variable name directly when passing
    # the parameter on the command-line
    method: str = Field("algo1", description="Algorithm to use.", flag_type="--")

    # For an actual parameter we would probably have a better name. Lets assume
    # This parameter (-p) modifies the behaviour of the method above.
    method_param1: Optional[int] = Field(
        description="Modify method performance.", flag_type="-", rename_param="p"
    )

    # We will only allow method to take on one of two values
    @validator("method")
    def validate_method(cls, method: str, values: Dict[str, Any]) -> str:
        """Method validator: --method can be algo1 or algo2."""

        valid_methods: List[str] = ["algo1", "algo2"]
        if method not in valid_methods:
            raise ValueError("method must be algo1 or algo2")
        return method

    # Lets change the default value of `method_param1` depending on `method`
    # NOTE: We didn't provide a default value to the Field above and made it
    # optional. We can use this to test whether someone is purposefully
    # overriding the value of it, and if not, set the default ourselves.
    # We set `always=True` since pydantic will normally not use the validator
    # if the default is not changed
    @validator("method_param1", always=True)
    def validate_method_param1(cls, param1: Optional[int], values: Dict[str, Any]) -> int:
        """method param1 validator"""

        # If someone actively defined it, lets just return that value
        # We could instead do some additional validation to make sure that the
        # value they provided is valid...
        if param1 is not None:
            return param1

        # method_param1 comes after method, so this will be defined, or an error
        # would have been raised.
        method: str = values['method']
        if method == "algo1":
            return 3
        elif method == "algo2":
            return 5
```

The special `root_validator(pre=False)` can also be used to provide validation of the model as a whole. This is also the recommended method for specifying a result (using `result_from_params`) which has a complex dependence on the parameters of the model. This latter use-case is described in FAQ 2 below.

#### FAQ
1. How can I specify a default value which depends on another parameter?

Use a custom validator. The example above shows how to do this. The parameter that depends on another parameter must come LATER in the model defintion than the independent parameter.

2. My `TaskResult` is determinable from the parameters model, but it isn't easily specified by one parameter. How can I use `result_from_params` to indicate the result?

When a result can be identified from the set of parameters defined in a `TaskParameters` model, but is not as straightforward as saying it is equivalent to one of the parameters alone, we can set `result_from_params` using a custom validator. In the example below, we have two parameters which together determine what the result is, `output_dir` and `out_name`. Using a validator we will define a result from these two values.

```py
from pydantic import Field, root_validator

class RunTask3Parameters(ThirdPartyParameters):
    """Parameters for the runtask3 binary."""

    class Config(ThirdPartyParameters.Config):
        set_result: bool = True       # This must be set here!
        result_from_params: str = ""  # We will set this momentarily

    # [...] executable, other params, etc.

    output_dir: str = Field(
        description="Directory to write output to.",
        flag_type="--",
        rename_param="dir",
    )

    out_name: str = Field(
        description="The name of the final output file.",
        flag_type="--",
        rename_param="oname",
    )

    # We can still provide other validators as needed
    # But for now, we just set result_from_params
    # Validator name can be anything, we set pre=False so this runs at the end
    @root_validator(pre=False)
    def define_result(cls, values: Dict[str, Any]) -> Dict[str, Any]:
        # Extract the values of output_dir and out_name
        output_dir: str = values["output_dir"]
        out_name: str = values["out_name"]

        result: str = f"{output_dir}/{out_name}"
        # Now we set result_from_params
        cls.Config.result_from_params = result

        # We haven't modified any other values, but we MUST return this!
        return values
```

3. My new `Task` depends on the output of a previous `Task`, how can I specify this dependency?
Parameters used to run a `Task` are recorded in a database for every `Task`. It is also recorded whether or not the execution of that specific parameter set was successful. A utility function is provided to access the most recent values from the database for a specific parameter of a specific `Task`. It can also be used to specify whether unsuccessful `Task`s should be included in the query. This utility can be used within a validator to specify dependencies. For example, suppose the input of `RunTask2` (parameter `input`) depends on the output location of `RunTask1` (parameter `outfile`). A validator of the following type can be used to retrieve the output file and make it the default value of the input parameter.

```py
from pydantic import Field, validator

from .base import ThirdPartyParameters
from ..db import read_latest_db_entry

class RunTask2Parameters(ThirdPartyParameters):
    input: str = Field("", description="Input file.", flag_type="--")

    @validator("input")
    def validate_input(cls, input: str, values: Dict[str, Any]) -> str:
        if input == "":
            task1_out: Optional[str] = read_latest_db_entry(
                f"{values['lute_config'].work_dir}",  # Working directory. We search for the database here.
                "RunTask1",                           # Name of Task we want to look up
                "outfile",                            # Name of parameter of the Task
                valid_only=True,                      # We only want valid output files.
            )
            # read_latest_db_entry returns None if nothing is found
            if task1_out is not None:
                return task1_out
        return input
```

There are more examples of this pattern spread throughout the various `Task` models.

### Specifying an `Executor`: Creating a runnable, "managed `Task`"

**Overview**

After a pydantic model has been created, the next required step is to define a **managed `Task`**. In the context of this library, a **managed `Task`** refers to the combination of an `Executor` and a `Task` to run. The `Executor` manages the process of `Task` submission and the execution environment, as well as performing any logging, eLog communication, etc. There are currently two types of `Executor` to choose from, **but only one is applicable to third-party code.** The second `Executor` is listed below for completeness only. If you need MPI see the note below.

1. `Executor`: This is the standard `Executor`. It should be used for third-party uses cases.
2. `MPIExecutor`: This performs all the same types of operations as the option above; however, it will submit your `Task` using MPI.

  - The `MPIExecutor` will submit the `Task` using the number of available cores - 1. The number of cores is determined from the physical core/thread count on your local machine, or the number of cores allocated by SLURM when submitting on the batch nodes.

**Using MPI with third-party `Task`s**

As mentioned, you should setup a third-party `Task` to use the first type of `Executor`. If, however, your third-party `Task` uses MPI this may seem non-intuitive. When using the `MPIExecutor` LUTE code is submitted with MPI. This includes the code that performs signalling to the `Executor` and `exec`s the third-party code you are interested in running. While it is possible to set this code up to run with MPI, it is more challenging in the case of third-party `Task`s because there is no `Task` code to modify directly! The `MPIExecutor` is provided mostly for first-party code. This is not an issue, however, since the standard `Executor` is easily configured to run with MPI in the case of third-party code.

When using the standard `Executor` for a `Task` requiring MPI, the `executable` in the pydantic model must be set to `mpirun`. For example, a third-party `Task` model, that uses MPI but is intended to be run with the `Executor` may look like the following. We assume this `Task` runs a Python script using MPI.

```py
class RunMPITaskParameters(ThirdPartyParameters):
    class Config(ThirdPartyParameters.Config):
        ...

    executable: str = Field("mpirun", description="MPI executable")
    np: PositiveInt = Field(
        max(int(os.environ.get("SLURM_NPROCS", len(os.sched_getaffinity(0)))) - 1, 1),
        description="Number of processes",
        flag_type="-",
    )
    pos_arg: str = Field("python", description="Python...", flag_type="")
    script: str = Field("", description="Python script to run with MPI", flag_type="")
```

**Selecting the `Executor`**

After deciding on which `Executor` to use, a single line must be added to the `lute/managed_tasks.py` module:

```py
# Initialization: Executor("TaskName")
TaskRunner: Executor = Executor("SubmitTask")
# TaskRunner: MPIExecutor = MPIExecutor("SubmitTask") ## If using the MPIExecutor
```

In an attempt to make it easier to discern whether discussing a `Task` or **managed `Task`**, the standard naming convention is that the `Task` (class name) will have a verb in the name, e.g. `RunTask`, `SubmitTask`. The corresponding **managed `Task`** will use a related noun, e.g. `TaskRunner`, `TaskSubmitter`, etc.

As a reminder, the `Task` name is the first part of the class name of the pydantic model, without the `Parameters` suffix. This name **must** match. E.g. if your pydantic model's class name is `RunTaskParameters`, the `Task` name is `RunTask`, and this is the string passed to the `Executor` initializer.

**Modifying the environment**

If your third-party `Task` can run in the standard `psana` environment with no further configuration files, the setup process is now complete and your `Task` can be run within the LUTE framework. If on the other hand your `Task` requires some changes to the environment, this is managed through the `Executor`. There are a couple principle methods that the `Executor` has to change the environment.

1. `Executor.update_environment`: if you only need to add a few environment variables, or update the `PATH` this is the method to use. The method takes a `Dict[str, str]` as input. Any variables can be passed/defined using this method. By default, any variables in the dictionary will overwrite those variable definitions in the current environment if they are already present, **except** for the variable `PATH`. By default `PATH` entries in the dictionary are prepended to the current `PATH` available in the environment the `Executor` runs in (the standard `psana` environment). This behaviour can be changed to either append, or overwrite the `PATH` entirely by an optional second argument to the method.
2. `Executor.shell_source`: This method will source a shell script which can perform numerous modifications of the environment (PATH changes, new environment variables, conda environments, etc.). The method takes a `str` which is the path to a shell script to source.

As an example, we will update the `PATH` of one `Task` and source a script for a second.

```py
TaskRunner: Executor = Executor("RunTask")
# update_environment(env: Dict[str,str], update_path: str = "prepend") # "append" or "overwrite"
TaskRunner.update_environment(
    { "PATH": "/sdf/group/lcls/ds/tools" }  # This entry will be prepended to the PATH available after sourcing `psconda.sh`
)

Task2Runner: Executor = Executor("RunTask2")
Task2Runner.shell_source("/sdf/group/lcls/ds/tools/new_task_setup.sh") # Will source new_task_setup.sh script
```

#### Parsing inputs and outputs: `tasklets`

Generally, in addition to the final output from the `Task`, we are also interested in some summary information. This could be a text summary of some key result in the larger output, or a graphical figure that describes the portions of interest.

For a third-party `Task` there isn't an easy way to insert code that provides these summaries into the `Task` itself, so the `tasklet` mechanism is provided. Tasklets are just Python functions. They are executed by the `Executor` either before or after the main `Task` has been run. One major use case of tasklets, is that the `Executor` can use the return values from these functions as the summary information for the main `Task` they are associated with. Tasklets are added to the `Executor` in much the same way that the execution environment is modified: using the `add_tasklet` method. E.g.

```py
Task3Runner: Executor = Executor("RunTask3")
Task3Runner.add_tasklet(
    callable,
    [arg1, arg2, arg3],
    when="before",
    set_result=False,
    set_summary=True
)
```

The `add_tasklet` method has the following signature:
```py
def add_tasklet(
    self,
    tasklet: Callable,
    args: Union[List[Any], Tuple[Any,...]],
    when: str,
    set_result: bool,
    set_summary: bool
)
```

These parameters, in order, are:

- `tasklet` : Any callable function. A number of tasklets are already defined in the `lute.tasks.tasklets` module as examples. Some of these are already associated with **managed** `Task`s.
- `args` : This is a list/tuple (or any iterable) of arguments to pass to the `tasklet`. The arguments can be substituted similarly to templates (below) or parameters in the configuration YAML. This is discussed further below.
- `when`: This is a string literal taking the values of `"before"` or `"after"`, indicating whether the tasklet function should be run before or after the actual `Task`, respectively.
- `set_result`: Is a bool. If `True`, the **main result payload** will be overwritten in the database with the return values of the tasklet. This affects only the database archiving - any actual files, etc., will not be overwritten. Regardless, in general this is not the appropriate option to use with a third-party `Task`.
- `set_summary`: Is a bool. If `True`, the **result summary** is set to the return values of the tasklet. This allows the main result of the `Task` to be recorded in addition to some auxiliary summary information. In general, we will want to use this option.

##### Substituting parameters for `tasklet` arguments

We often want the input to a tasklet to depend on some of the `TaskParameters` for the main associated `Task`. We can specify this using a Jinja-like substitution syntax. When passing arguments to the `add_tasklet` method, we can enclose the name of parameters from the `TaskParameters` model in a string between doubly curly brackets: `"{{ param_to_sub }}"`. For example if we wanted to use the output file as the input to the tasklet, assuming it had the parameter name `out_file`, we would use `"{{ out_file }}"`.

You can substitute multiple parameters in every string if necessary, with each parameter to substitute enclosed in its own set of double curly brackets.

**Note:** The parameter substitutions must be passed as strings. Type conversions will be performed to the actual type of the parameter you are substituting.

##### Return types and associated actions

The `Executor` will decide on what to do with a returned value from a tasklet based upon the specific type. Note that these types can also be used in first-party `Task`s to perform the same actions; however, in that case, users should set the `_result.summary` or `_result.payload` directly, rather than using a tasklet (if possible).

The following types and actions are currently defined:

- `Dict[str, str]`: Post key/value pairs under the report section in the control tab of the eLog. These key/values are also posted as run parameters if possible. This return type is best used for short text summaries, e.g. indexing rate, execution time, etc. The key/value pairs are converted to a semi-colon delimited string for storage in the database. E.g. `{"Rate": 0.05, "Total": 10}` will be stored as `Rate: 0.05;Total: 10` in the database.
- `ElogSummaryPlots`: This special dataclass, defined in `lute.tasks.dataclasses` is used to create an eLog summary plot under the Summaries tab. The path to the created HTML file is stored in the database.

You can return any number of these objects from a tasklet as a tuple. Each item will be processed independently. For datbase archiving, the various entries are stored semi-colon delimited.

##### Examples

Some example `tasklets` are available in `lute.tasks.tasklets`. Some of these are reproduced below.

**Generic Usage**

- `concat_files(location: str, in_files_glob: str, out_file: str)`: Concatenate a set of output files
- `grep(match_str: str, in_file: str)`: Search for occurences of a string in some output file.
- `git_clone(repo: str, location: str, permissions: str)`: Clone a git repository. This is generally of more use as a tasklet run **before** the main `Task` is run.

**Specific Examples**

- `indexamajig_summary_indexing_rate`: Calculates indexing rate based on parsing a stream file.
- `compare_hkl_fom_summary`: Extracts a figure of merit **and** produces a summary plot.

Refer to `managed_tasks` to see how these are specifically called with `CrystFELIndexer` and `HKLComparer` respectively.



### Using templates: managing third-party configuration files

Some third-party executables will require their own configuration files. These are often separate JSON or YAML files, although they can also be bash or Python scripts which are intended to be edited. Since LUTE requires its own configuration YAML file, it attempts to handle these cases by using Jinja templates. When wrapping a third-party task a template can also be provided - with small modifications to the `Task`'s pydantic model, LUTE can process special types of parameters to render them in the template. LUTE offloads all the template rendering to Jinja, making the required additions to the pydantic model small. On the other hand, it does require understanding the Jinja syntax, and the provision of a well-formatted template, to properly parse parameters. Some basic examples of this syntax will be shown below; however, it is recommended that the `Task` implementer refer to the [official Jinja documentation](https://jinja.palletsprojects.com/en/3.1.x/) for more information.

**Note:** By default templated parameters are **NOT** validated, i.e., type-checked. This default case is handled first below. If knowledgeable about appropriate typing for the template variables, further modification of the `TaskParameters` model is possible to include validation. This advanced use-case is described second.

#### Non-validated template parameters

LUTE provides two additional base models which are used for template parsing in conjunction with the primary `Task` model. These are:

- `TemplateParameters` objects which hold parameters which will be used to render a portion of a template.
- `TemplateConfig` objects which hold two strings: the name of the template file to use and the full path (including filename) of where to output the rendered result.

`Task` models which inherit from the `ThirdPartyParameters` model, as all third-party `Task`s should, allow for extra arguments. LUTE will parse any extra arguments provided in the configuration YAML as `TemplateParameters` objects automatically, which means that they do not need to be explicitly added to the pydantic model (although they can be). As such the **only** requirement on the Python-side when adding template rendering functionality to the `Task` is the addition of one parameter - an instance of `TemplateConfig`. The instance **MUST** be called `lute_template_cfg`.

```py
from pydantic import Field, validator

from .base import TemplateConfig

class RunTaskParamaters(ThirdPartyParameters):
    ...
    # This parameter MUST be called lute_template_cfg!
    lute_template_cfg: TemplateConfig = Field(
        TemplateConfig(
            template_name="name_of_template.json",
            output_path="/path/to/write/rendered_output_to.json",
        ),
        description="Template rendering configuration",
    )
```

LUTE looks for the template in `config/templates`, so only the name of the template file to use within that directory is required for the `template_name` attribute of `lute_template_cfg`. LUTE can write the output anywhere (the user has permissions), and with any name, so the full absolute path including filename should be used for the `output_path` of `lute_template_cfg`.

The rest of the work is done by the combination of Jinja, LUTE's configuration YAML file, and the template itself. Understanding the interplay between these components is perhaps best illustrated by an example. As such, let us consider a simple third-party `Task` whose only input parameter (on the command-line) is the location of a configuration JSON file. We'll call the third-party executable `jsonuser` and our `Task` model, the `RunJsonUserParameters`. We assume the program is run like:
```bash
jsonuser -i <input_file.json>
```

The first step is to setup the pydantic model as before.

```py
from pydantic import Field, validator

from .base import TemplateConfig

class RunJsonUserParameters:
    executable: str = Field(
        "/path/to/jsonuser", description="Executable which requires a JSON configuration file."
    )
    # Lets assume the JSON file is passed as "-i <path_to_json>"
    input_json: str = Field(
        "", description="Path to the input JSON file.", flag_type="-", rename_param="i"
    )
```

The next step is to create a template for the JSON file. Let's assume the JSON file looks like:
```
{
    "param1": "arg1",
    "param2": 4,
    "param3": {
        "a": 1,
        "b": 2
    },
    "param4": [
        1,
        2,
        3
    ]
}
```

Any, or all of these values can be substituted for, and we can determine the way in which we will provide them. I.e. a substitution can be provided for each variable individually, or, for example for a nested hierarchy, a dictionary can be provided which will substitute all the items at once. For this simple case, let's provide variables for `param1`, `param2`, `param3.b` and assume that we want the first and second entries for `param4` to be identical for our use case (i.e., we can use one variable for them both. In total, this means we will perform 5 substitutions using 4 variables. Jinja will substitute a variable anywhere it sees the following syntax, `{{ variable_name }}`. As such a valid template for our use-case may look like:
```
{
    "param1": {{ str_var }},
    "param2": {{ int_var }},
    "param3": {
        "a": 1,
        "b": {{ p3_b }}
    },
    "param4": [
        {{ val }},
        {{ val }},
        3
    ]
}
```

We save this file as `jsonuser.json` in `config/templates`. Next, we will update the original pydantic model to include our template configuration. We still have an issue, however, in that we need to decide where to write the output of the template to. In this case, we can use the `input_json` parameter. We will assume that the user will provide this, although a default value can also be used. A custom validator will be added so that we can take the `input_json` value and update the value of `lute_template_cfg.output_path` with it.

```py
# from typing import Optional

from pydantic import Field, validator

from .base import TemplateConfig #, TemplateParameters

class RunJsonUserParameters:
    executable: str = Field(
        "jsonuser", description="Executable which requires a JSON configuration file."
    )
    # Lets assume the JSON file is passed as "-i <path_to_json>"
    input_json: str = Field(
        "", description="Path to the input JSON file.", flag_type="-", rename_param="i"
    )
    # Add template configuration! *MUST* be called `lute_template_cfg`
    lute_template_cfg: TemplateConfig = Field(
        TemplateConfig(
            template_name="jsonuser.json", # Only the name of the file here.
            output_path="",
        ),
        description="Template rendering configuration",
    )
    # We do not need to include these TemplateParameters, they will be added
    # automatically if provided in the YAML
    #str_var: Optional[TemplateParameters]
    #int_var: Optional[TemplateParameters]
    #p3_b: Optional[TemplateParameters]
    #val: Optional[TemplateParameters]


    # Tell LUTE to write the rendered template to the location provided with
    # `input_json`. I.e. update `lute_template_cfg.output_path`
    @validator("lute_template_cfg", always=True)
    def update_output_path(
        cls, lute_template_cfg: TemplateConfig, values: Dict[str, Any]
    ) -> TemplateConfig:
        if lute_template_cfg.output_path == "":
            lute_template_cfg.output_path = values["input_json"]
        return lute_template_cfg
```

All that is left to render the template, is to provide the variables we want to substitute in the LUTE configuration YAML. In our case we must provide the 4 variable names we included within the substitution syntax (`{{ var_name }}`). The names in the YAML must match those in the template.

```yaml
RunJsonUser:
    input_json: "/my/chosen/path.json" # We'll come back to this...
    str_var: "arg1" # Will substitute for "param1": "arg1"
    int_var: 4 # Will substitute for "param2": 4
    p3_b: 2  # Will substitute for "param3: { "b": 2 }
    val: 2 # Will substitute for "param4": [2, 2, 3] in the JSON
```

If on the other hand, a user were to have an already valid JSON file, it is possible to turn off the template rendering. (ALL) Template variables (`TemplateParameters`) are simply excluded from the configuration YAML.

```yaml
RunJsonUser:
    input_json: "/path/to/existing.json"
    #str_var: ...
    #...
```

#### Validated template parameters

If you are able to provide validation for template parameters this is preferred, although it is not always straightforward to determine appropriate parameter types/validators. LUTE provides a custom validator which can be used in conjunction with a separate pydantic model for the template parameters to provide type-checking.

By way of example, we will re-write the model above for the `RunJsonUser` `Task` in order to validate the template parameters. We begin by creating a new pydantic `BaseModel` to hold all the template parameters. This class can be defined anywhere, but for organizational purposes the class is often defined within the `TaskParameters` class.

```py
# Import BaseModel if not present - required for validation!
from pydantic import BaseModel

class RunJsonUserParameters:

    class RunJsonTemplateParameters(BaseModel):
        # If you want to allow extra, un-validated parameters include this
        # Config as well.
        # class Config(BaseModel.Config):
        #     extra: str = "allow"

        str_var: Optional[str] = Field(
            None, description="This string does..."
        )
        int_var: Optional[int] = Field(
            None, description="This int does..."
        )
        p3_b: Optional[int] = Field(
            None, description="This parameter does..."
        )
        val: Optional[int] = Field(
            None, description="This parameter does..."
        )
```

Next the template parameters defined in the class need to be made available through a validated parameter within the `TaskParameters` class. We will import a special validator defined in `lute.io.models.validators` in order to perform the necessary options to handle the individual template parameters during validation.

```py
# Import BaseModel if not present - required for validation!
from pydantic import BaseModel

# Import custom validators for template parameters!
from lute.io.models.validators import template_parameter_validator

class RunJsonUserParameters:

    class RunJsonTemplateParameters:
        # If you want to allow extra, un-validated parameters include this
        # Config as well.
        # class Config(BaseModel.Config):
        #     extra: str = "allow"

        str_var: Optional[str] = Field(
            None, description="This string does..."
        )
        int_var: Optional[int] = Field(
            None, description="This int does..."
        )
        p3_b: Optional[int] = Field(
            None, description="This parameter does..."
        )
        val: Optional[int] = Field(
            None, description="This parameter does..."
        )
    # Define the validator - the argument must match the parameter name!
    _set_template_parameters = template_parameter_validator("json_parameters")

    # executable...
    # input_json...
    json_parameters: Optional[RunJsonTemplateParameters] = Field(
        None, description="Optional template parameters..."
    )
```

The rest of the model remains unchanged. However, we **do** need to make a change to how we pass the template parameters to LUTE through the configuration YAML. Previously, we provided each template parameter (`str_var`, `int_var`, ...) as an individual parameter in the YAML file. Now, they must be passed as a dictionary under the key `json_parameters`, as this is the parameter we have defined to hold template parameters in the model.

```yaml
RunJsonUser:
    input_json: "/my/chosen/path.json" # We'll come back to this...
    json_parameters:
        str_var: "arg1" # Will substitute for "param1": "arg1"
        int_var: 4 # Will substitute for "param2": 4
        p3_b: 2  # Will substitute for "param3: { "b": 2 }
        val: 2 # Will substitute for "param4": [2, 2, 3] in the JSON
```

Now, the individual parameters will be validated according to the model definition (`RunJsonTemplateParameters`). As previously, if we do not want to use any template parameters, we simply remove them from the YAML, although in this case, we must remove the full section beginning with `json_parameters`. As we've used the same parameter names as previously, our Jinja template does not need to change

#### Additional Jinja Syntax
There are many other syntactical constructions we can use with Jinja. Some of the useful ones are:

**If Statements** - E.g. only include portions of the template if a value is defined.
```
{% if VARNAME is defined %}
// Stuff to include
{% endif %}
```

**Loops** - E.g. Unpacking multiple elements from a dictionary.
```
{% for name, value in VARNAME.items() %}
// Do stuff with name and value
{% endfor %}
```

## Creating a "First-Party" `Task`
The process for creating a "First-Party" `Task` is very similar to that for a "Third-Party" `Task`, with the difference being that you must also write the analysis code. The steps for integration are:
1. Write the `TaskParameters` model.
2. Write the `Task` class. There are a few rules that need to be adhered to.
3. Make your `Task` available by modifying the import function.
4. Specify an `Executor`

### Specifying a `TaskParameters` Model for your `Task`
Parameter models have a format that must be followed for "Third-Party" `Task`s, but "First-Party" `Task`s have a little more liberty in how parameters are dealt with, since the `Task` will do all the parsing itself.

To create a model, the basic steps are:

1. If necessary, create a new module (e.g. `new_task_category.py`) under `lute.io.models`, or find an appropriate pre-existing module in that directory.
  - An `import` statement must be added to `lute.io.models._init_` if a new module is created, so it can be found.
  - If defining the model in a pre-existing module, make sure to modify the `__all__` statement to include it.
2. Create a new model that inherits from `TaskParameters`. You can look at `lute.models.io.tests.TestReadOutputParameters` for an example. **The model must be named** `<YourTaskName>Parameters`

  - You should include **all** relevant parameters here, including input file, output file, and any potentially adjustable parameters. These parameters **must** be included even if there are some implicit dependencies between `Task`s and it would make sense for the parameter to be auto-populated based on some other output. Creating this dependency is done with validators (see step 3.). All parameters should be overridable, and all `Task`s should be fully-independently configurable, based solely on their model and the configuration YAML.
  - To follow the preferred format, parameters should be defined as: `param_name: type = Field([default value], description="This parameter does X.")`
3. Use validators to do more complex things for your parameters, including populating default values dynamically:

  - E.g. create default values that depend on other parameters in the model - see for example: [SubmitSMDParameters](https://github.com/slac-lcls/lute/blob/57f2a0889ec9603e3b8642f485c27df7d1f6e96f/lute/io/models/smd.py#L139).
  - E.g. create default values that depend on other `Task`s by reading from the database - see for example: [TestReadOutputParameters](https://github.com/slac-lcls/lute/blob/57f2a0889ec9603e3b8642f485c27df7d1f6e96f/lute/io/models/tests.py#L75).
4. The model will have access to some general configuration values by inheriting from `TaskParameters`. These parameters are all stored in `lute_config` which is an instance of `AnalysisHeader` ([defined here](https://github.com/slac-lcls/lute/blob/57f2a0889ec9603e3b8642f485c27df7d1f6e96f/lute/io/models/base.py#L42)).

  - For example, the experiment and run number can be obtained from this object and a validator could use these values to define the default input file for the `Task`.

A number of configuration options and **Field** attributes are also available for "First-Party" `Task` models. These are identical to those used for the `ThirdPartyTask`s, although there is a smaller selection. These options are reproduced below for convenience.

**Config settings and options**
Under the class definition for `Config` in the model, we can modify global options for all the parameters. In addition, there are a number of configuration options related to specifying what the outputs/results from the associated `Task` are, and a number of options to modify runtime behaviour. Currently, the available configuration options are:

| **Config Parameter** | **Meaning**                                                                                                  | **Default Value**     | **ThirdPartyTask-specific?**             |
|:--------------------:|:------------------------------------------------------------------------------------------------------------:|:---------------------:|:----------------------------------------:|
| `run_directory`      | If provided, can be used to specify the directory from which a `Task` is run.                                | `None` (not provided) | **NO**                                   |
| `set_result`         | `bool`. If `True` search the model definition for a parameter that indicates what the result is.             | `False`               | **NO**                                   |
| `result_from_params` | If `set_result` is `True` can define a result using this option and a validator. See also `is_result` below. | `None` (not provided) | **NO**                                   |
| `short_flags_use_eq` | Use equals sign instead of space for arguments of `-` parameters.                                            | `False`               | **YES** - Only affects `ThirdPartyTask`s |
| `long_flags_use_eq`  | Use equals sign instead of space for arguments of `-` parameters.                                            | `False`               | **YES** - Only affects `ThirdPartyTask`s |
|                      |                                                                                                              |                       |                                          |

These configuration options modify how the parameter models are parsed and passed along on the command-line, as well as what we consider results and where a `Task` can run. The default behaviour is that parameters are assumed to be passed as `-p arg` and `--param arg`, the `Task` will be run in the current working directory (or scratch if submitted with the ARP), and we have no information about `Task` results . Setting the above options can modify this behaviour.

- By setting `short_flags_use_eq` and/or `long_flags_use_eq` to `True` parameters are instead passed as `-p=arg` and `--param=arg`.
- By setting `run_directory` to a valid path, we can force a `Task` to be run in a specific directory. By default the `Task` will be run from the directory you submit the job in, or from your scratch folder (`/sdf/scratch/...`) if you submit from the eLog. Some `ThirdPartyTask`s rely on searching the correct working directory in order run properly.
- By setting `set_result` to `True` we indicate that the `TaskParameters` model will provide information on what the `TaskResult` is. This setting must be used with one of two options, either the `result_from_params` `Config` option, described below, or the **Field** attribute `is_result` described in the next sub-section (**Field Attributes**).
- `result_from_params` is a Config option that can be used when `set_result==True`. In conjunction with a **validator** (described a sections down) we can use this option to specify a result from all the information contained in the model. E.g. if you have a `Task` that has parameters for an `output_directory` and a `output_filename`, you can set `result_from_params==f"{output_directory}/{output_filename}"`.


**Field attributes**
In addition to the global configuration options there are a couple of ways to specify individual parameters. The following `Field` attributes are used when parsing the model:

| **Field Attribute** | **Meaning**                                                                                            | **Default Value** | **Example**                                       |
|:-------------------:|:------------------------------------------------------------------------------------------------------:|:-----------------:|:-------------------------------------------------:|
| `description`       | Documentation of the parameter's usage or purpose.                                                     | N/A               | `arg = Field(..., description="Argument for...")` |
| `is_result`         | `bool`. If the `set_result` `Config` option is `True`, we can set this to `True` to indicate a result. | N/A               | `output_result = Field(..., is_result=true)`      |
|                     |                                                                                                        |                   |                                                   |

### Writing the `Task`
You can write your analysis code (or whatever code to be executed) as long as it adheres to the limited rules below. You can create a new module for your `Task` in `lute.tasks` or add it to any existing module, if it makes sense for it to belong there. The `Task` itself is a single class constructed as:

1. Your analysis `Task` is a class named in a way that matches its Pydantic model. E.g. `RunTask` is the `Task`, and `RunTaskParameters` is the Pydantic model.
2. The class must inherit from the `Task` class (see template below). **If you intend to use MPI see the following section.**
3. You must provide an implementation of a `_run` method. This is the method that will be executed when the `Task` is run. You can in addition write as many methods as you need. For fine-grained execution control you can also provide `_pre_run()` and `_post_run()` methods, but this is optional.
4. For all communication (including print statements) you should use the `_report_to_executor(msg: Message)` method. Since the `Task` is run as a subprocess this method will pass information to the controlling `Executor`. You can pass **any** type of object using this method, strings, plots, arrays, etc.
5. If you did not use the `set_result` configuration option in your parameters model, make sure to provide a result when finished. This is done by setting `self._result.payload = ...`. You can set the result to be any object. If you have written the result to a file, for example, please provide a path.

A minimal template is provided below.

```py
"""Standard docstring..."""

__all__ = ["RunTask"]
__author__ = "" # Please include so we know who the SME is

# Include any imports you need here

from lute.execution.ipc import Message # Message for communication
from lute.io.models.base import *      # For TaskParameters
from lute.tasks.task import *          # For Task

class RunTask(Task): # Inherit from Task
    """Task description goes here, or in __init__"""

    def __init__(self, *, params: TaskParameters) -> None:
        super().__init__(params=params) # Sets up Task, parameters, etc.
        # Parameters will be available through:
          # self._task_parameters
          # You access with . operator: self._task_parameters.param1, etc.
        # Your result object is availble through:
          # self._result
            # self._result.payload <- Main result
            # self._result.summary <- Short summary
            # self._result.task_status <- Semi-automatic, but can be set manually

    def _run(self) -> None:
        # THIS METHOD MUST BE PROVIDED
        self.do_my_analysis()

    def do_my_analysis(self) -> None:
        # Send a message, proper way to print:
        msg: Message(contents="My message contents", signal="")
        self._report_to_executor(msg)

        # When done, set result - assume we wrote a file, e.g.
        self._result.payload = "/path/to/output_file.h5"
        # Optionally also set status - good practice but not obligatory
        self._result.task_status = TaskStatus.COMPLETED
```

#### Using MPI for your `Task`

In the case your `Task` is written to use `MPI` a slight modification to the template above is needed. Specifically, an additional keyword argument should be passed to the base class initializer: `use_mpi=True`. This tells the base class to adjust signalling/communication behaviour appropriately for a multi-rank MPI program. Doing this prevents tricky-to-track-down problems due to ranks starting, completing and sending messages at different times. The rest of your code can, as before, be written as you see fit. The use of this keyword argument will also synchronize the start of all ranks and wait until all ranks have finished to exit.

```py
"""Task which needs to run with MPI"""

__all__ = ["RunTask"]
__author__ = "" # Please include so we know who the SME is

# Include any imports you need here

from lute.execution.ipc import Message # Message for communication
from lute.io.models.base import *      # For TaskParameters
from lute.tasks.task import *          # For Task

# Only the init is shown
class RunMPITask(Task): # Inherit from Task
    """Task description goes here, or in __init__"""

    # Signal the use of MPI!
    def __init__(self, *, params: TaskParameters, use_mpi: bool = True) -> None:
        super().__init__(params=params, use_mpi=use_mpi) # Sets up Task, parameters, etc.
        # That's it.
```

#### Message signals

Signals in `Message` objects are strings and can be one of the following:

```py
LUTE_SIGNALS: Set[str] = {
    "NO_PICKLE_MODE",
    "TASK_STARTED",
    "TASK_FAILED",
    "TASK_STOPPED",
    "TASK_DONE",
    "TASK_CANCELLED",
    "TASK_RESULT",
}
```
Each of these signals is associated with a hook on the `Executor`-side. They are for the most part used by base classes; however, you can choose to make use of them manually as well.

### Making your `Task` available
Once the `Task` has been written, it needs to be made available for import. Since different `Task`s can have conflicting dependencies and environments, this is managed through an import function. When the `Task` is done, or ready for testing, a condition is added to `lute.tasks.__init__.import_task`. For example, assume the `Task` is called `RunXASAnalysis` and it's defined in a module called `xas.py`, we would add the following lines to the `import_task` function:

```py
# in lute.tasks.__init__

# ...

def import_task(task_name: str) -> Type[Task]:
    # ...
    if task_name == "RunXASAnalysis":
        from .xas import RunXASAnalysis

        return RunXASAnalysis
```

### Defining an `Executor`
The process of `Executor` definition is identical to the process as described for `ThirdPartyTask`s above. The one exception is if you defined the `Task` to use MPI as described in the section above (Using MPI for your `Task`), you will likely consider using the `MPIExecutor`.

#### Environment setup
Currently, first-party `Task`s are expected to use the same environment as the `Executor`, so while you can use the environment update methods to insert environment variables, complete replacement of the environment is not supported. If you have a compelling reason to require this feature, contact the maintainers.

#### `tasklet` usage
You can also use `tasklet` functions with first-party `Task`s if needed. The preferred method, however, would be to incorporate whatever tasklet code is needed directly into your `Task`.
