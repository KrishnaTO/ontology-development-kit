# Instructions for DEVELOPERs of ODK

This is intended for developers only, see [README](README.md) for the
main docs.

## Installation

For running locally without Docker you will need

 * robot
 * owltools
 * python3.6 or higher

See [Dockerfile](Dockerfile) for details on how to obtain these

## How it works

Previously ODK used a perl script to create a new repo. This iterated
the [template/](template) directory and used special magic for expanding into a
target folder. This has been replaced by python code
[odk/odk.py](odk/odk.py) with makes used of Jinja2 templates.

For example, the file
[template/src/ontology/Makefile.jinja2](template/src/ontology/Makefile.jinja2)
will compile to a file `src/ontology/Makefile` in the target/output
directory.

Jinja2 templates should be fairly easy to grok for anyone familiar
with templating systems. The syntax is very similar to Liquid
templates, which are used extensively on the OBO site. We feed the
template engine with a project object that is passed in by the user
(more on that later).

Logic in the templates should be non-existent.

## Dynamic File Names

Sometimes the odk needs to create a file whose name is based on an
input setting or configuration; sometimes lists of such files need to
be created.

For example, if the user specifies 3 external ontology dependencies,
then we want to see the repo with 3 files `imports/{{ont.id}}_import.owl`

Rather than embed this logic in code, we include all dynamic files in
a single "tar-esque" formated file: [template/_dynamic_files.jinja2](template/_dynamic_files.jinja2)

This file is actually a specification for multiple files, each target
file specified with `^^^`. Because the parent file is interpreted
using templates, we can have dynamic file names, and entire files
created via looping constructs.

## The Project object

Currently the datamodel is specified as python dataclasses, for now
the best way to see the complete spec is to look at the classes
annotated with `@dataclass` in the code.

There is a [schema](schema) folder but this is incomplete as the
dataclasses-scheme module doesn't appear to work (TODO)...

There are also example `project.yaml` files in the
[examples](examples) folder, and these also serve as rudimentary unit
tests.

See for example [examples/go-mini/project.yaml](examples/go-mini/project.yaml)

The basic data model is:

 * An `OntologyProject` consists of various configuration settings, plus `ProductGroup`s
 * These are:
    * An `ImportProduct` group which specifies how import files are generated
    * A `SubsetProduct` group which specifies how subset/slim files are generated
    * Other product groups for reports and templates

Many ontology projects need only specify a very minimal configuration:
id of ontology, github/gitlab location, and list of ontology ids for
imports. However, for projects that need to customize there are
multiple options. E.g. for an import product you can optionally
specific a particular URL that overrides the default PURL.

Note that for backwards compatibility, a project.yaml file is not
required. A user can specify an entire repo by running `seed` with
options such as `-d` for dependencies.

Note that in all cases a `project.yaml` file is generated.

## ODK commands

```
$ ./odk/odk.py --help
Usage: odk.py [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  create-dynfile   For testing purposes
  create-makefile  For testing purposes
  dump-schema      Dumps the python schema as json schema.
  export-project   For testing purposes
  seed             Seeds an ontology project
```

The most common command is seed.

## Updating a Makefile and/or repo

Previously with odk there was no path to either upgrading an existing
project with new settings (i.e. adding an import) OR to take advantage
of changes to the odk (e.g changes in the core Makefile).

This should now be easier with the new odk, although the
implementation emphasis has been on the seed command. Some things that
will make this easier:

 * Convention of using a second loaded Makefile for custom changes
 * Maintaining a project.yaml in root folder will allow easy regeneration

TODO: add a refresh command. This could run odk *in place*, but
preserving protected files. TBD how to determine protected
files. Obviously the edit file should not be touched. Could use git
log to determine if any modifications have been made?

## Setting up a new machine for ODK development

In order to build and publish ODK, you need the following:

1. `docker` installed
2. have `git` installed
3. have a git user.name and user.email set

## General SOP for ODK release and publication

- There are three types of releases: major, minor and development snapshot.
  - Major versions include changes to the workflow system.
  - Minor versions include changes to tools, such as ROBOT or Python dependencies.
  - Development snapshots reflect the current state of the `main` (`master`) branch.
- They all have slightly different procedures which we will detail below.

### Major releases

Major releases contain changes to the workflow system of the ODK, e.g. changes to the `Makefile` and various supporting scripts (e.g. run.sh, update_repo.sh).
They require users to update their repository with `sh run.sh make update_repo`
Major releases are typically incremented (a bit confusingly) on the "minor" version number of ODK, i.e. 1.4, 1.5, 1.6 etc.
There are currently (2024) no plans to increment on the major version - this will likely be reserved to fundamental changes like switching from `make` to another workflow system or dropping `docker` (both are unlikely to happen in the midterm).
There should be no more than 2 such version updates per year (ideally 1), to reduce the burden on users to maintain their repositories.

#### SOP for creating a major release

* Put the `master` branch in the state we want for release (i.e. merge any approved PR that we want included in that release, etc.).
* Ensure your local `master` branch is up-to-date (`git pull`) and run a basic build (`make build tests`). This _should_ not result in any surprises as this exact command is run every time we merge a change into the `master` branch by our CI system. However, as various dependencies of the system are still variable (in particular unix package versions), there are occassionally situations where the build fails or, less likely, the subsequent tests.
* Do any amount of testing as needed to be confident we are ready for release. For major releases, it makes sense to test the ODK on at least 10 ontologies. In 2024 we typically test:
  * All ontologies we test for _minor_ releases (see below)
  * Flybase ontologies ([fbbt](https://github.com/FlyBase/drosophila-anatomy-developmental-ontology), [fbcv](https://github.com/FlyBase/drosophila-developmental-ontology), [dpo](https://github.com/FlyBase/drosophila-phenotype-ontology))
  * [NCBITaxon](https://github.com/obophenotype/ncbitaxon)
  * [Zebrafish Phenotype Ontology](https://github.com/obophenotype/zebrafish-phenotype-ontology) (should only be done in collaboration with a ZP core developer, too many points of failure)
* We suggest to have at least 1 other ODK core team member run 3 release pipelines to reduce the risk of operating system related differences.
* Run `docker login` to ensure you are logged in. You must have access rights to `obolibrary` organisation to run the following.
* Run `docker buildx create --name multiarch --driver docker-container --use` _if you have not done so in the past_. NOTE: This command needs to be run only once. Its effects are persistent, so it will never be needed again for any subsequent release — unless you completely reset your Docker installation in the meantime.
* Run `make publish-multiarch` to publish the ODK in the `obolibrary` dockerhub organisation (see [below](#multi-arch-images) for details).
* OPTIONAL: If you want publish the multi-arch images under the `obotools/` organisation, you need to run locally:
  ```sh
  $ docker buildx create --name multiarch --driver docker-container --use
  $ make publish-multiarch IM=obotools/odkfull IMLITE=obotools/odklite DEV=obotools/odkdev
  ```  
* Immediately when the release is finished, create and publish a GitHub release (check last major release on how to format correctly).
* After the release is _published_, create a new PR updating the `VERSION = "v1.X"` variable in the `Makefile` to the next major version number.

### Minor releases

Minor releases are normally releases that contain only changes about the _tools_ provided by the ODK, and no changes about the _workflows_. As such, they do not require users to update their repositories. All users need to do to start using a new minor release is to pull the latest Docker image of the ODK (`pull obolibrary/odkfull:latest`).

Minor releases are only provided for the current major branch of the ODK. For example, if the latest major release is v1.5, we will provide (as needed) minor releases v1.5.1, v1.5.2, etc, but we will _not_ provide minor releases for any version prior to 1.5; once v1.6 is released, we will likewise stop providing v1.5.x minor releases. In other words, only one major branch is actively supported at any time.

#### SOP for creating a minor release

* As soon as a major branch (v1.X) has been released, create a `BRANCH-1.X-MAINTENANCE` branch forked from the `v1.X` release tag.
* As development of the next major branch (v1.X+1) is ongoing, routinely backport tools-related changes to the `BRANCH-1.X-MAINTENANCE` branch.
* By convention, changes to the next major branch that are introduced by a PR tagged with a `hotfix` label should also be backported to the maintenance branch.
* To avoid cluttering the maintenance branch with multiple “Python constraints update” backport commits, it is recommended to backport all Python constraints at once, shortly before a minor release.
* There are no strict guidelines about when a minor release should happen. The availability of a new version of ROBOT is usually reason enough to make such a release, but upgrades to other tools can also occasionally justify a minor release.

Once the decision to make a minor release has been made:

* Make sure all tools-related updates (including Python tools) have been backported.
* Do any amount of testing as needed to be confident we are ready for release. For minor releases, it makes sense to test the ODK on at least 5 ontologies. In 2024 we typically test:
  - [Mondo](https://github.com/monarch-initiative/mondo) ([docs](https://mondo.readthedocs.io/en/latest/developer-guide/release/)) (a lot of use of old tools, like owltools, interleaved with ROBOT, heavy dependencies on serialisations, perl scripts)
  - [Mondo Ingest](https://github.com/monarch-initiative/mondo-ingest) (a lot of use of sssom-py and OAK, interleaved with heavyweight ROBOT pipelines)
  - [Uberon](https://github.com/obophenotype/uberon) (ROBOT plugins, old tools like owltools)
  - [Human Phenotype Ontology](https://github.com/obophenotype/human-phenotype-ontology) (uses of ontology translation system (babelon), otherwise pretty standard ODK, high impact ontology)
  - [Cell Ontology](https://github.com/obophenotype/cell-ontology) (Relatively standard, high impact ODK setup)
* Update the CHANGELOG.md file.
* Bump the version number to `v1.X.Y` in
  - the top-level `Makefile`,
  - the `Makefile` in the `docker/odklite` directory.
* If the minor release includes a newer version of ROBOT, and if that has not already been done when ROBOT itself was updated, update the version number in `docker/robot/Makefile` so it matches the version of ROBOT that is used.
* Push all last-minute changes (CHANGELOG and version number updates) to the `BRANCH-1.X-MAINTENANCE` branch.
* Build and publish the images from the tip of the `BRANCH-1.X-MAINTENANCE` branch (same procedure as above to build and publish a major release).
* Create a GitHub release from the tip of the `BRANCH-1.X-MAINTENANCE` branch, with a `v1.X.Y` tag.
* Resume backporting changes to the `BRANCH-1.X-MAINTENANCE` until the time comes for the next minor release.

### Development snapshot

Development snapshots reflect the current state of the main (`master`) branch. They do not undergo the same level of testing (or any testing at all) as the normal releases, and are intended to help trialing and debugging the changes that happen in the `master` branch.

Development snapshots should _not_ be used in a production environment. Feel free to use them if you want to help us developing the next major release, but if you use them in your production pipelines, understand that you’re doing so at your own risk.

Development snapshots are tagged with the `dev` tag on docker, and with the `-dev` suffix in the `Makefile` pipeline (e.g. `v1.6-dev` to indicate that this is a snapshot of the ODK on the way towards a 1.6 release). Development snapshots can happen any time, but typically happen once every 1 to 4 weeks.

#### SOP for creating a development snapshot

* Put the `master` branch in the state we want for release (i.e. merge any approved PR that we want included in that release, etc.).
* Ensure your local `master` branch is up-to-date (`git pull`) and run a basic build (`make build tests`) (see comments in Major release section for details about the rationale).
* We do not typically do any additional testing for the development snapshot.
* Run `docker login` to ensure you are logged in. You must have access rights to `obolibrary` organisation to run the following.
* Run `docker buildx create --name multiarch --driver docker-container --use` _if you have not done so in the past_. NOTE: This command needs to be run only once. Its effects are persistent, so it will never be needed again for any subsequent release — unless you completely reset your Docker installation in the meantime.
* Run `make publish-multiarch-dev` to publish the ODK in the `obolibrary` dockerhub organisation (see [below](#multi-arch-images) for details).
* Do NOT create a GitHub release!
* Your build has been successful when the `dev` image appears as updated [on Dockerhub](https://hub.docker.com/r/obolibrary/odkfull/tags).

## Docker

Note that with v1.2 the main odkfull [Dockerfile](Dockerfile) is at
the root level. We now use a base alpine image for compactness, and
selectively add in unix tools like make and rsync.

Note also that we include odk.py and the template folders in the
image. This means that odk seed can now be run from anywhere!

To build the Docker image from the top level:

```
make build
```

Note that this means local invocations to use `obolibrary/odkfull`
will use the version you built.

To test:

```
make tests
```

To publish on Dockerhub:

```
make publish
```

<a id="multi-arch-images"></a>

### Multi-arch images

To build multi-arch images that will work seemleassly on several
platforms, you need to have [buildx](https://github.com/docker/buildx)
enabled on your Docker installation. On MacOS with Docker Desktop,
`buildx` should already be enabled. For other systems, refer to Docker's
documentation.

Create a *builder* instance for multi-arch builds (this only needs to be
done once):

```
docker buildx create --name multiarch --driver docker-container --use
```

You can then build and push multi-arch images by running:

```
make publish-multiarch
```

Use the variable `PLATFORMS` to specify the architectures for which an
image should be built. The default is `linux/amd64,linux/arm64`, for
images that work on both x86_64 and arm64 machines.

To publish only the development version:

```
make publish-multiarch-dev
```

Sometimes, it may be necessary to delete the multiarch and redo it (roughly once per month):

```
docker buildx rm multiarch
docker buildx create --name multiarch --driver docker-container --use
```

## Some notes on templating and logic

There is a potential for some confusion as to responsibility for
logic. On the one hand we have dependency logic in the Makefile. But
we also have minimal logic in deciding what to put in the Makefile.

For example, we could move some logic from the Makefile by using
for/endfor Jinja constructs and unfolding every product in a group and
have an explicit non-pattern target in the Makefile. Or we can
continue to write targets with patterns. Or we can do a mixture of
both.

Additionally there is some minimal logic in the python odk code, but
this is kept to an absolute minimum; the role of the python code is to
run template expansions.

In general the decision is to keep the templating as simple as
possible, which leads to a slight mixed two level system.

One gotcha is the two levels of comments. The `{# .. #}` comments are
template comments for the eyes of developers only. These are ignored
when compiling down to the target file. Then we also have Makefile
comments `#` which remain in the target file, and are intended for
advanced ontology maintainers who need to debug their workflows. These
are intermingled in Makefile.jinja2

## Unit Tests

To run:

```
make test
```

These will seed a few example repos in the target/ folder, some from command line opts, others from a project.yaml

These are pseudo-tests as the output is not examined, however they do
serve to guard against multiple kinds of errors as the seed script
will often fail if things are not set up correctly.

The examples folder serves for both unit test and documentation purposes.

## Migration System

TODO

## Pull request rules

1. One PR per feature.
2. Each PR must link to one or more existing issues.
3. There should be one commit per logical change. This is important so we can be more effective at cherry picking for patch releases.
4. Every commit should have an appropriate title and description.

## Major minor system

- Once a release is done, we work on the master branch towards the next major release
- The minor release steward cherry picks certain kinds of changes for minor releases, including
   - ROBOT updates
   - Python tool version updates
   - Critical bug fixes 

## Adding new programs or Python modules to the ODK

How and where to add a component to the ODK depends on the nature of the
component and whether it is to be added to `odkfull` or `odklite`.

As a general rule, new components should probably be added to `odkfull`,
as `odklite` is intended to be kept small. Components should only be
added to `odklite` if they are required in rules from the ODK-generated
standard Makefile. Note that any component added to `odklite` will
automatically be part of `odkfull`.

Is the component available as a standard Ubuntu package? Then add it to
the list of packages in the `apt-get install` invocation in [the main
Dockerfile](Dockerfile) (for inclusion into `odkfull`) or in [the
Dockerfile for odklite](docker/odklite/Dockerfile).

Is the component available as a pre-built binary? Be careful that many
projets only provide pre-built binaries for the x86 architecture. Using
such a binary would result in the component being unusable in the arm64
version of the ODK (notably used on Apple computers equipped with M1
CPUs, aka "Apple Silicon").

Java programs available as pre-built jars can be installed by adding new
`RUN` commands at the end of either the main Dockerfile (for `odkfull`)
or the Dockerfile for `odklite`.

If the component needs to be built from source, do so in [the Dockerfile for odkbuild](docker/builder/Dockerfile), 
and install the compiled file(s) in either the `/staging/full` tree or the `/staging/lite` tree, for
inclusion in `odkfull` or `odklite` respectively.

If the component is a Python package, add it to the `requirements.txt.full`
file, and *also* in the `requirements.txt.lite` file if it is to be part
of `odklite`. Please try to avoid version constraints unless you can
explain why you need one.

Python packages are "frozen" so that any subsequent build of the ODK
will always include the exact same version of every single package. To
update the frozen list, run `make constraints.txt` in the top-level
directory. This should be done at least (1) whenever a new package is
added to `requirements.txt.full`, (2) whenever the base image is
updated. It can also be done at any time during the development cycle to
ensure that we pick regular updates of any package we use.
