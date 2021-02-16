---
title: GitHub Actions
---

{% note info %}
# {% fa fa-graduation-cap %} What you'll learn

- How to run Cypress tests with GitHub Actions as part of CI/CD pipeline
- How to parallelize Cypress test runs within GitHub Actions

{% endnote %}

GitHub offers developers {% url "Actions" https://github.com/features/actions %} that provide a way to **automate, customize, and execute your software development workflows** within your GitHub repository.  Detailed documentation is available in the {% url "GitHub Action Documentation" https://docs.github.com/en/actions %}.

# Cypress GitHub Action

Workflows can be packaged and shared as {% url "GitHub Actions" https://github.com/features/actions %}.  GitHub maintains many, such as the {% url "checkout" https://github.com/marketplace/actions/checkout %} and {% url "cache" https://github.com/marketplace/actions/cache %} actions used below.

The Cypress team maintains the official {% url "Cypress GitHub Action" https://github.com/marketplace/actions/cypress-io %} for running Cypress tests. This action provides npm installation, custom caching, additional configuration options and simplifies setup of advanced workflows with Cypress in the GitHub Actions platform.

# Basic Setup

The example below is basic CI setup and job using the {% url "Cypress GitHub Action" https://github.com/marketplace/actions/cypress-io %} to run Cypress tests within the Electron browser. This GitHub Action configuration is placed within `.github/workflows/main.yml`.

```yaml
name: Cypress Tests

on: [push]

jobs:
  cypress-run:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
      # Install NPM dependencies, cache them correctly
      # and run all Cypress tests
      - name: Cypress run
        uses: cypress-io/github-action@v2
        with:
          build: npm run build
          start: npm start
```

{% note success Try it out%}
To try out the example above yourself, fork the {% url "Cypress Kitchen Sink" https://github.com/cypress-io/cypress-example-kitchensink %} example project and place the above GitHub Action configuration in `.github/workflows/main.yml`.
{% endnote %}

**How this action works:**

- On *push* to this repository, this job will provision and start GitHub-hosted Ubuntu Linux instance for running the outlined `steps` for the declared `cypress-run` job within the `jobs` section of the configuration.
- The {% url "GitHub checkout Action" https://github.com/marketplace/actions/checkout %} is used to checkout our code from our GitHub repository.
- Finally, our Cypress GitHub Action will:
  - Install npm dependencies
  - Build the project (`npm run build`)
  - Start the project web server (`npm start`)
  - Run the Cypress tests within our GitHub repository within Electron.

# Testing in Chrome and Firefox with Cypress Docker Images

GitHub Actions provides the option to specify a container image for the job. Cypress offers various {% url "Docker Images" https://github.com/cypress-io/cypress-docker-images %} for running Cypress locally and in CI.

Below we add the `container` attribute using a {% url "Cypress Docker Image" https://github.com/cypress-io/cypress-docker-images %} built with Google Chrome and Firefox. For example, this allows us to run the tests in Firefox by passing the `browser: firefox` attribute to the {% url "Cypress GitHub Action" https://github.com/marketplace/actions/cypress-io %}.

```yaml
name: Cypress Tests using Cypress Docker Image

on: [push]

jobs:
  cypress-run:
    runs-on: ubuntu-latest
    container: cypress/browsers:node12.18.3-chrome87-ff82
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      # Install NPM dependencies, cache them correctly
      # and run all Cypress tests
      - name: Cypress run
        uses: cypress-io/github-action@v2
        with:
          # Specify Browser since container image is compile with Firefox
          browser: firefox
```

# Caching Dependencies and Build Artifacts

Dependencies and build artifacts maybe cached between jobs using the {% url "cache" https://github.com/marketplace/actions/cache %} GitHub Action.

The job below includes a cache of `node_modules`, the Cypress binary in `~/.cache/Cypress` and the `build` directory.  In addition, the `build` attribute is added to the Cypress GitHub Action to generate the build artifacts prior to the test run.

{% note info %}
Caching of dependencies and build artifacts can be accomplished with the GitHub Actions {% url " Upload/Download Artifact Actions" https://docs.github.com/en/actions/guides/storing-workflow-data-as-artifacts %}, but uploading takes more time than using the {% url "cache GitHub Action" https://github.com/marketplace/actions/cache %}.
{% endnote %}

```yaml
name: Cypress Tests with Dependency and Artifact Caching

on: [push]

jobs:
  cypress-run:
    runs-on: ubuntu-latest
    container: cypress/browsers:node12.18.3-chrome87-ff82
    steps:
      - name: Checkout
        uses: actions/checkout@v2
      
      - uses: actions/cache@v2
        id: yarn-build-cache
        with:
          path: |
            node_modules
            ~/.cache/Cypress
            build
          key: ${{ runner.os }}-node_modules-build-${{ hashFiles('**/yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-node_modules-build-

      # Install NPM dependencies, cache them correctly
      # and run all Cypress tests
      - name: Cypress run
        uses: cypress-io/github-action@v2
        with:
          # Specify Browser since container image is compile with Firefox
          browser: firefox
          build: yarn build
      
```

# Parallelization

The {% url "Cypress Dashboard" 'dashboard' %} offers the ability to {% url 'parallelize and group test runs' parallelization %} along with additional insights and {% url "analytics" analytics %} for Cypress tests.

GitHub Actions offers a {% url "matrix strategy" https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstrategymatrix %} for declaring different job configurations for a single job definition. Jobs declared within a matrix strategy can run in parallel which enables us run multiples instances of Cypress at same time as we will see later in this section.

Before diving into an example of a parallelization setup, it is important to understand the two different types of GitHub Action jobs that we will declare:
- **Install Job**: A job that installs and caches dependencies that will used by subsequent jobs later in the GitHub Action workflow.
- **Worker Job**: A job that handles execution of Cypress tests and depends on the *install job*. 

## Install Job

The separation of installation from test running is necessary when running parallel jobs. It allows for reuse of various build steps aided by caching.

First, we'll define the `install` step that will be used by the worker jobs defined in the matrix strategy.

For the `steps`, notice that we pass `runTests: false` to the Cypress GitHub Action to instruct it to only install dependencies *without running the tests*.

The {% url "cache" https://github.com/marketplace/actions/cache %} GitHub Action is included and will save the state of the `node_modules`, `~/.cache/Cypress` and `build` directories for the worker jobs.

```yaml
name: Cypress Tests with installation job

on: [push]

jobs:
  install:
    runs-on: ubuntu-latest
    container: cypress/browsers:node12.18.3-chrome87-ff82
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - uses: actions/cache@v2
        id: yarn-and-build-cache
        with:
          path: |
            ~/.cache/Cypress
            build
            node_modules
          key: ${{ runner.os }}-node_modules-build-${{ hashFiles('**/yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-node_modules-build-

      - name: Cypress install
        uses: cypress-io/github-action@v2
        with:
          runTests: false
          build: yarn build
```

## Worker Jobs
{% note info %}
The following configuration with `parallel` and `record` options to the {% url "Cypress GitHub Action" https://github.com/marketplace/actions/cypress-io %} requires a subscription to the {% url "Cypress Dashboard" https://on.cypress.io/dashboard %}.
{% endnote %}

Next we define the worker jobs.

Below we define a job for "UI Chrome Tests" that will run tests under the `cypress/tests/ui/` path against Google Chrome.

It has a {% url "matrix strategy" https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstrategymatrix %} for 5 workers.

{% note info %}
We must specify the `container` attribute using a {% url "Cypress Docker Image" https://github.com/cypress-io/cypress-docker-images %} in the worker configuration that was used in the install job.
{% endnote %}

{% note info %}
The {% url "checkout" https://github.com/marketplace/actions/checkout %} and {% url "cache" https://github.com/marketplace/actions/cache %} actions must be declared in each job as there is no state between the `install` job and other jobs (e.g. `ui-chrome-tests`) except what is persisted by cache.
{% endnote %}

Using our {% url "Cypress GitHub Action" https://github.com/marketplace/actions/cypress-io %} we specify `install: false` since our dependencies and build were cache in our `install` job.

Finally, we tell the Cypress GitHub Actions to record results to the {% url "Cypress Dashboard" https://on.cypress.io/dashboard %} (using the `CYPRESS_RECORD_KEY` environment variable) in parallel.


Jobs can be organized by groups and in this job we specify a `group: "UI - Chrome"` to consolidate all runs for these workers in a central location in the {% url "Cypress Dashboard" https://on.cypress.io/dashboard %}.

```yaml
name: Cypress Tests with Install Job and UI Chrome Job x 5

on: [push]

jobs:
  install:
  # ...

  ui-chrome-tests:
    runs-on: ubuntu-latest
    container: cypress/browsers:node12.18.3-chrome87-ff82
    needs: install
    strategy:
      fail-fast: false
      matrix:
        # run copies of the current job in parallel
        containers: [1, 2, 3, 4, 5]
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - uses: actions/cache@v2
        id: yarn-and-build-cache
        with:
          path: |
            ~/.cache/Cypress
            build
            node_modules
          key: ${{ runner.os }}-node_modules-build-${{ hashFiles('**/yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-node_modules-build-

      - name: "UI Tests - Chrome"
        uses: cypress-io/github-action@v2
        with:
          # we have already installed all dependencies above
          install: false
          start: yarn start:ci
          wait-on: "http://localhost:3000"
          wait-on-timeout: 120
          browser: chrome
          record: true
          parallel: true
          group: "UI - Chrome"
          spec: cypress/tests/ui/*
        env:
          CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}
          # Recommended: pass the GitHub token lets this action correctly
          # determine the unique run id necessary to re-run the checks
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```


{% note info %}
#### {% fa fa-graduation-cap %} Real World Example

A complete CI workflow against multiple browsers, viewports and operating systems is available in the {% url "Real World App (RWA)" https://github.com/cypress-io/cypress-realworld-app %}.

Clone the {% fa fa-github %} {% url "Real World App (RWA)" https://github.com/cypress-io/cypress-realworld-app %} and refer to the {% url ".github/workflows/main.yml" https://github.com/cypress-io/cypress-realworld-app/blob/develop/github/workflows/main.yml %} file.

{% imgTag /img/guides/github-actions/rwa-run-matrix.png "Cypress Real World App GitHub Actions Matrix" "no-border" %}
{% endnote %}
