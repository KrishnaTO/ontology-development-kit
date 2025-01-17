# ODK review

ODK is a core part of the Ontology development and management workflow, consisting of a variety of services and libraries within a pipeline. Currently, it is based on Docker to set up for its base OS (ubuntu), services (Dokerfile > '# Install tools provided by Ubuntu.'), and libraries (Makefile > requirements.txt.full).

See CONTRIBUTING.md for development readme

## Workflows

There is some discussion regarding the components within this repository here: https://github.com/INCATools/ontology-development-kit/issues/877. It breaks down the repo into 3 workflows:

1. Toolbox: docker image
2. Template: jinja2 / cookiecutter
3. odk-runner tool: seed-via-docker.{sh/bat}

### 2. Templates

Templates are based on jinja2 templates, initiated by {./odk/odk.py} to use:

> For example, the file
> [template/src/ontology/Makefile.jinja2](template/src/ontology/Makefile.jinja2)
> will compile to a file `src/ontology/Makefile` in the target/output
> directory.

Intializing an Ontology uses

> example `project.yaml` files in the
> [examples](examples) folder

## Requirements

These list of requirements have intricately been worked on, since >7 years, and the current list of modules requires review.

### Files inventory

    ./docker/
        |_ builder/
            |_ Dockerfile: `odkbuild` image, solely used as staging area to build `odklite` and `odkfull` images
            |_ Makefile
        |_ odklite/
            |_ Dockerfile:  `odkfull` lighter variant, subset of programs and Python modules found in the full ODK
            |_ Makefile
    ./odk/
        |_ odk.py: copied to '/work' in docker container to run `odk-pipeline`
    ./template/: jinja2 templates for base ontology, [odk/odk.py](odk/odk.py) makes use of these Jinja2 templates
    ./Dockerfile: `odkfull`, "main" image, requires `odklite` image (./docker/odklite/Dockerfile)
    ./odk.sh: script maps the CD into docker container at /work, to review settings(?)

    ./Makefile: running tests on the complete ODK package on travis; users don't need to use
    ./update-constraints.sh
    ./requirements.txt.full
    ./requirements.txt.lite
    ./constraints.txt
    ./pip-constraints.txt
    
    ./seed-via-docker.bat: seed new ontology
    ./seed-via-docker.sh: seed new ontology

#### Readmes with info
docker\README.md
CONTRIBUTING.md

Ideally, an automated graph consisting of relationships would assist in establishing workflows.
For instance:

- Makefile: "for running tests on the complete ontology-development-kit package on travis; users should not need to use this"
    -> update-constraints.sh: updates the constraint requirements
        -> requirements.txt
            -> requirements.txt.full
        -> pip-requirements.txt

 - seed-via-docker.bat: Windows-version of initializing ODK Ontology template via docker image
|
 - seed-via-docker.sh: create ODK Ontology template via docker image, per README.md#Customizing your ODK installation
    - Dockerfile: manually specifies Base OS and services; installs requirements per module/service (ex. Jena, obographviz, OBO-Dashboard, etc)


### Docker templates:
- Builder image {./docker/builder/Dockerfile}: build everything that we cannot install in pre-compiled form
- ODK Lite {./docker/odklite/Dockerfile}
    -> copy {repo}/odk/odk.py {dockerimage}/tools/odk.py
    - Final ODK image {./Dockerfile}: built upon the odklite image
