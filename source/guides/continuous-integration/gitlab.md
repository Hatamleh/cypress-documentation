---
title: GitLab CI/CD
---

{% note info %}
# {% fa fa-graduation-cap %} What you'll learn

- How to run Cypress tests with GitLab as part of CI/CD pipeline
- How to parallelize Cypress test runs within GitLab CI/CD

{% endnote %}

With it's hosted {% url "CI/CD Service" https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/ %}, {% url "GitLab" https://gitlab.com %} offers developers "a tool built into GitLab for software development through the {% url "continuous methodologies" https://docs.gitlab.com/ee/ci/introduction/index.html#introduction-to-cicd-methodologies %}".

Detailed documentation is available in the {% url "GitLab CI/CD Documentation" https://docs.gitlab.com/ee/ci/introduction/ %}.

# Basic Setup

The example below is basic CI setup and job using {% url "GitLab CI/CD" https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/ %} to run Cypress tests within the Electron browser. This GitHub Action configuration is placed within `.gitlab-ci.yml`.

```yaml
stages:
  - test

test:
  image: node:latest
  stage: test
  script:
    # install dependencies
    - npm ci
    # start the server in the background
    - npm run start:ci &
    # run Cypress tests
    - npm run e2e
```

{% note info Try it out %}
To try out the example above yourself, fork the {% url "Cypress Kitchen Sink" https://github.com/cypress-io/cypress-example-kitchensink %} example project and place the above GitHub Action configuration in `.gitlab-ci.yml`.
{% endnote %}


**How this configuration works:**

- On *push* to this repository, this job will provision and start GitLab-hosted Linux instance for running the outlined `stages` declared in `script` with in the `test` job section of the configuration.
- The code is checked out from our GitHub/GitLab repository.
- Finally, our scripts will:
  - Install npm dependencies
  - Start the project web server (`npm start`)
  - Run the Cypress tests within our GitHub repository within Electron.

# Testing in Chrome and Firefox with Cypress Docker Images

The Cypress team maintains the official {% url "Docker Images" https://github.com/cypress-io/cypress-docker-images %} for running Cypress tests locally and in CI, which are built with Google Chrome and Firefox. For example, this allows us to run the tests in Firefox by passing the `--browser firefox` attribute to `cypress run`.

```yaml
stages:
  - test

test:
  image: cypress/browsers:node12.14.1-chrome85-ff81
  stage: test
  script:
    # install dependencies
    - npm ci
    # start the server in the background
    - npm run start:ci &
    # run Cypress tests
    - npx cypress run --browser firefox 
```

# Caching Dependencies and Build Artifacts

Per the {% url "GitLab documentation" https://docs.gitlab.com/ee/ci/caching/ %}, {% url "GitLab CI/CD" https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/ %} offers options for caching dependencies and build artifacts across many different workflows.

To cache `node_modules`, the npm cache across builds, the `cache` attribute and configuration has been added below.

Artifacts from a job can be defined by providing paths and an optional expiry time.

```yaml
stages:
  - test

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .npm/

test:
  image: cypress/browsers:node12.14.1-chrome85-ff81
  stage: test
  script:
    # install dependencies
    - npm ci
    # start the server in the background
    - npm run start:ci &
    # run Cypress tests
    - npx cypress run --browser firefox 
  artifacts:
    when: always
    paths:
      - cypress/videos/**/*.mp4
      - cypress/screenshots/**/*.png
    expire_in: 1 day
```
# Parallelization

The {% url "Cypress Dashboard" 'dashboard' %} offers the ability to {% url 'parallelize and group test runs' parallelization %} along with additional insights and {% url "analytics" analytics %} for Cypress tests.

{% note info %}
The following configuration with `--parallel` and `--record` options to Cypress requires a subscription to the {% url "Cypress Dashboard" https://on.cypress.io/dashboard %}.
{% endnote %}

Our command records results to the {% url "Cypress Dashboard" https://on.cypress.io/dashboard %} in parallel, using the `CYPRESS_RECORD_KEY` environment variable.

Jobs can be organized by groups by passing a `--group` attribute and value to `cypress run`.

Below we pass the `parallel` attribute with a numerical value for the number of workers.

The results from each worker will be consolidated into the group name in the {% url "Cypress Dashboard" https://on.cypress.io/dashboard %}.


```yaml
stages:
  - test

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .npm/

test:
  image: cypress/browsers:node12.14.1-chrome85-ff81
  stage: test
  parallel: 5
  script:
    # install dependencies
    - npm ci
    # start the server in the background
    - npm run start:ci &
    # run Cypress tests in parallel
    - npx cypress run --record --parallel --browser firefox --group "UI - Firefox"
  artifacts:
    when: always
    paths:
      - cypress/videos/**/*.mp4
      - cypress/screenshots/**/*.png
    expire_in: 1 day
```

{% note info %}
#### {% fa fa-graduation-cap %} Real World Example

A complete CI workflow against multiple browsers, viewports and operating systems is available in the {% url "Real World App (RWA)" https://github.com/cypress-io/cypress-realworld-app %}.

Clone the {% fa fa-github %} {% url "Real World App (RWA)" https://github.com/cypress-io/cypress-realworld-app %} and refer to the {% url ".gitlab-ci.yml" https://github.com/cypress-io/cypress-realworld-app/blob/develop/.gitlab-ci.yml %} file.
{% endnote %}

# Debugging with the Cypress Dashboard