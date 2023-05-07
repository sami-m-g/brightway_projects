# bw_projects

[![PyPI](https://img.shields.io/pypi/v/bw_projects.svg)][pypi status]
[![Status](https://img.shields.io/pypi/status/bw_projects.svg)][pypi status]
[![Python Version](https://img.shields.io/pypi/pyversions/bw_projects)][pypi status]
[![License](https://img.shields.io/pypi/l/bw_projects)][license]

[![Read the documentation at https://bw_projects.readthedocs.io/](https://img.shields.io/readthedocs/bw_projects/latest.svg?label=Read%20the%20Docs)][read the docs]
[![Tests](https://github.com/brightway-lca/bw_projects/actions/workflows/python-test.yml/badge.svg)][tests]
[![Codecov](https://codecov.io/gh/brightway-lca/bw_projects/branch/main/graph/badge.svg?token=ZVWBCITI4A)][codecov]

[![pre-commit](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit&logoColor=white)][pre-commit]
[![Black](https://img.shields.io/badge/code%20style-black-000000.svg)][black]

[pypi status]: https://pypi.org/project/bw_projects/
[read the docs]: https://bw_projects.readthedocs.io/
[tests]: https://github.com/brightway-lca/bw_projects/actions?workflow=Tests
[codecov]: https://codecov.io/gh/brightway-lca/bw_projects
[pre-commit]: https://github.com/pre-commit/pre-commit
[black]: https://github.com/psf/black

This is a library to manage subdirectories, so that work on a project can be isolated from other projects. It is designed for use with the [Brightway life cycle assessment](https://brightway.dev/) software framework, but has no dependencies on Brightway and can be used independently.

Project metadata is stored in SQLite using the [Peewee ORM](http://docs.peewee-orm.com/en/latest/). The SQLite file is in the `base_directory`, and project data is stored in subdirectories. By default, [platformdirs](https://github.com/platformdirs/platformdirs) is used to create the `base_directory`, though this can be overridden.

## Installation

Via pip or conda (`conda-forge` channel).

Depends on:

* [peewee](http://docs.peewee-orm.com/en/latest/)
* [platformdirs](https://github.com/platformdirs/platformdirs)
* [python_slugify](https://github.com/un33k/python-slugify)

## Usage

### Initializing the ProjectsManager

```python
from bw_projects import ProjectsManager  # This doesn't create anything yet

# This gets default config using `platformdirs` and initializes directories and database
projects_manager = ProjectsManager()
```

### Overriding defaults

If you want to use `platformdirs`, but with a different app configuration than the default (`app_name`: "Brightway3", `app_author`: "pycla"), pass a `Configuration` class instance:

```python
from bw_projects import Configuration, ProjectsManager

config = Configuration(
    app_name="something",
    app_author="else",
)
projects_manager = ProjectsManager(config=config)
```

See the `config.Configuration` class for other input options.

You can also directly specify the directory used for the projects database and the project directories:

```python
from bw_projects import ProjectsManager

projects_manager = ProjectsManager(
    dir_base="<path/to/your/base/directory>",
    database_name="my_projects.db"
)
```

Note that setting either `dir_base` or using a non-default `Configuration` will create a new projects database file, as the this database file is stored in the base directory of the project data subdirectories.

### Callbacks

This library does not provide a lot of functionality, and one often wants to
perform additional actions when projects are created, activated, or deleted. You
can register callback functions for each of these actions, and they all take the
same input arguments:

```python
def callback_function(manager, name, attributes, absolute_directory_path):
    print(f"Doing something with project {name} in {absolute_directory_path}")
```

Where:

* `manager` is the instance of `ProjectsManager` calling the callback
* `name` is a string
* `attributes` is a dictionary
* `absolute_directory_path` is a `pathlib.Path` instance

It's best to specify callback functions when the `ProjectsManager` is
instantiated. The input arguments are:

* `callbacks_create_project`
* `callbacks_activate_project`
* `callbacks_delete_project`

The callbacks are called **after** their respective `ProjectsManager` functions.
If you need to do something with the project data directory before it is deleted,
use `ProjectsManager.delete_project(name="something", delete_dir=False)`.

Here is an example of a callback:

```python
from bw_projects import ProjectsManager
from pathlib import Path

def check_awesomeness(
    manager: ProjectsManager,
    name: str,
    attributes: dict,
    absolute_directory_path: Path
):
    if attributes.get("awesome"):
        print("Activated awesome project")
    else:
        print(f"{name} isn't awesome; there are {len(manager)} projects, try again")

projects_manager = ProjectsManager(callbacks_activate_project=[check_awesomeness])
```

### Project creation

```python
projects_manager.create_project("<project_name>")
```

Creating a project does not activate it.

`ProjectsManager.create_project` takes the following keyword arguments:

* name: str. Name of project
* attributes: dict. Additional attributes of the project.
* exists_ok: bool, default `False`. Will raise `bw_projects.errors.ProjectExistsError` if this is false and the project exists.
* activate: bool, default `False`. Activate project after creation.

### Project activation

```python
projects_manager.activate_project("<project_name>")
```

Activating a project sets `projects_manager.active_project`, and calls the functions in `projects_manager.callbacks_activate_project`.

### Project deletion

```python
projects_manager.delete_project("<project_name>")
```

`ProjectsManager.delete_project` takes the following keyword arguments:

* name: str. Name of project to delete
* not_exist_ok: bool, default `False`. Will raise `peewee.DoesNotExist` if this is false and the project doesn't exist.
* delete_dir: bool, default `True`. Delete the project data directory.

Deleting the current activate project will set the `active_project` to `None`.

### Other project functionality

A few other miscellaneous functions are supported, including:

* `len(projects_manager)`
* `repr(projects_manager)`
* `"foo" in projects_manager`
* `iter(projects_manager)` - yields instances of `model.Project`

### Differences with previous Brightway functionality

* Instantiating the `ProjectsManager` class doesn't create a project by default.
* There is no equivalent to `set_project`, which would be create and activate a project. This now requires two actions.
* There is no soft deleting of projects by default; keeping the project data directory requires `delete_dir=False`.
* The way we create safe strings for project data directories has changed to using [python-slugify](https://github.com/un33k/python-slugify).
* We don't check environment variables to determine directory path. Those advanced enough to use this functionality can write a wrapper for `ProjectsManager` that sets `dir_base` reading whatever environment variable or config file they want.
* Iterating over the projects returns `bw_projects.model.Project` instances, not project names as strings.
* Deleting the current project will not automatically switch to another project.

## Contributing

Contributions are very welcome.
To learn more, see the [Contributor Guide][Contributor Guide].

## License

Distributed under the terms of the [GPL 3.0 license][License],
_bw_projects_ is free and open source software.

## Issues

If you encounter any problems,
please [file an issue][Issue Tracker] along with a detailed description.

<!-- github-only -->

[License]: https://github.com/brightway-lca/bw_projects/blob/main/LICENSE
[Contributor Guide]: https://github.com/brightway-lca/bw_projects/blob/main/CONTRIBUTING.md
[Issue Tracker]: https://github.com/brightway-lca/bw_projects/issues
