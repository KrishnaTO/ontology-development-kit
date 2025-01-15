# ODK review

ODK is a core part of the Ontology development and management workflow, consisting of a variety of services and libraries within a pipeline. Currently, it is based on Docker to set up for its base OS (ubuntu), services (Dokerfile > '# Install tools provided by Ubuntu.'), and libraries (Makefile > requirements.txt.full).

These list of requirements have intricately been worked on, since >7 years, and the current list of modules requires review.

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
