# poetry-gitlab

A container image for testing poetry-based projects in GitLab CI.

[![Docker Repository on Quay](https://quay.io/repository/rmcgover/poetry-gitlab/status "Docker Repository on Quay")](https://quay.io/repository/rmcgover/poetry-gitlab)

## Overview

This repository builds a container image to `quay.io/rmcgover/poetry-gitlab:latest`.

The image is intended for use in GitLab CI to test Python-based projects using [poetry](https://python-poetry.org/).

Installed software in the image includes:

- OS: latest stable Fedora (rebuilt automatically as base image is updated)
- tox
- poetry
- git
- [glab](https://github.com/profclems/glab) (GitLab CLI)
- [tkn](https://github.com/tektoncd/cli) (Tekton CLI)

## Example

Here's an example of using the image in a GitLab CI pipeline. We use
`poetry` to update poetry.lock, then `glab` to create a merge commit
with the result.

```yaml
"Poetry: update lock file":
  image:
    name: quay.io/rmcgover/poetry-gitlab:latest
  cache:
    paths:
      - .cache/pypoetry
  tags: [docker]
  variables:
    # poetry caching works OK if we point poetry at a path underneath
    # $CI_PROJECT_DIR.
    POETRY_CACHE_DIR: $CI_PROJECT_DIR/.cache/pypoetry

  script:

    # update poetry.lock
    - poetry update --vv --no-interaction --lock

    # commit the result
    - git add poetry.lock
    - git commit -m "Updated dependencies"

    # push it
    - git push $GIT_REMOTE_URL_VAR +HEAD:refs/heads/poetry-update
    
    # make a merge request
    - >-
        glab mr create
        --create-source-branch
        --remove-source-branch
        --repo $CI_PROJECT_PATH
        --head $CI_PROJECT_PATH
        --source-branch poetry-update
        -d "Automated update of dependencies."
        --title "chore: update poetry.lock"
        -l poetry,chore
        --no-editor
        -b $CI_COMMIT_BRANCH
        --yes

  variables:
    # The following vars affect the behavior of 'glab' tool.
    # POETRY_UPDATE_TOKEN should be a protected/masked CI/CD var holding an access
    # token for the current project.
    GITLAB_TOKEN: "$POETRY_UPDATE_TOKEN"
    GITLAB_HOST: "https://$CI_SERVER_HOST"
    NO_PROMPT: "1"
    NO_COLOR: "1"
    GIT_REMOTE_URL_VAR: "https://token:$POETRY_UPDATE_TOKEN@$CI_SERVER_HOST/$CI_PROJECT_PATH.git"
  rules:
    - if: >-
        $POETRY_UPDATE_TOKEN &&
        ($CI_PIPELINE_SOURCE == "web" || $CI_PIPELINE_SOURCE == "schedule") &&
        $POETRY_UPDATE_LOCKFILE == "1"
```

## License

This program is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.
