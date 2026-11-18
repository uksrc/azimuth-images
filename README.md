# azimuth-images

This repository contains [Packer](https://www.packer.io/) template definitions
for building images for use with [Azimuth](https://github.com/azimuth-cloud/azimuth-images).

The templates work by provisioning machines in an [OpenStack](https://www.openstack.org/)
project and configuring them using [Ansible](https://www.ansible.com/) before
converting to an image. These images are then downloaded from OpenStack and uploaded
to S3 for distribution in an Azimuth release.

## Pull upstream changes

Keep the repository up date by regularly pulling and merging upstream changes.

Make sure upstream is set to github.com/azimuth-cloud/azimuth-images:

```sh
git remote -v

origin git@github.com:uksrc/azimuth-images.git (fetch)
origin git@github.com:uksrc/azimuth-images.git (push)
upstream https://github.com/azimuth-cloud/azimuth-images (fetch)
upstream https://github.com/azimuth-cloud/azimuth-images (push)
```

If upstream is not set add it using:

```sh
git remote add upstream https://github.com/azimuth-cloud/azimuth-images

git remote -v

git fetch upstream
```

Check the upstream releases and decide which tag you want to merge e.g. 0.38.0. Create a branch for the changes and merge in the tag:

```sh
git checkout -b pog-123-merge-upstream-release-0.38.0
git merge 0.38.0
```

Carefully check all the changes and resolve any conflicts, being careful to retain local edits where appropriate.

When you are happy commit the changes, push the branch and create an MR. Test images will be automatically built by the pipeline for MRs that are ready for review i.e. not draft. Clean out any old/unused images that are no longer required.

You can also create an image manually.

## Building an image manually

Before building an image, you must configure OpenStack authentication, using either
an RC file or `clouds.yaml`, e.g.:

```sh
export OS_CLOUD=openstack
```

This repository has a concept of "environments", allowing builds to be customised for
different clouds if required, for example in a fork. These manifest as directories
under `env`, where environment are set that in turn can specify Packer var files
under `var`.

Currently, the repository contains a `base` environment with cloud-agnostic configuration
and an `arcus` environment that contains configuration for the continuous integration
environment, which is kindly hosted on the Arcus cloud at the University of Cambridge.

To build images manually, outside of CI, first ensure you have Packer installed and
the required providers installed:

```sh
packer init ./config.pkr.hcl
```

Next, install [uv](https://docs.astral.sh/uv/) and use it to create a virtual
environment and install the Python and Ansible dependencies:

```sh
uv sync
source .venv/bin/activate

ansible-galaxy install -f -r requirements.yml
```

Finally, set up an environment using the `arcus` environment for inspiration.

You can then build images using the `./bin/build-image` script. This script expects the
following environment variables to be set:

- `ENVIRONMENT` - the environment to use
- `PACKER_TEMPLATE` - the Packer template in `packer` to use
- `ENV_VAR_FILES` - the environment variable files to include

The values for `PACKER_TEMPLATE` and `ENV_VAR_FILES` can be determined by looking at
[the defined builds](./.github/builds.yaml).

For example, to build a workstation image, the following command can be used:

```sh
ENVIRONMENT=myenv \
PACKER_TEMPLATE=linux-desktop \
ENV_VAR_FILES=common,kvm,linux,ubuntu-jammy \
./bin/build-image
```

This will result in a private image being created in the target OpenStack project.

## Developing locally

To run the GitHub Actions linters locally, use:

```sh
docker run --rm \
    -e RUN_LOCAL=true \
    --env-file "super-linter.env" \
    -v "$(pwd)":/tmp/lint \
    ghcr.io/super-linter/super-linter:v7.3.0
```

```sh
ansible-lint -c .ansible-lint.yml ansible/
```

To run the tests and type checks for the Python scripts in `bin/`, use:

```sh
uv run pytest
uv run mypy
```
