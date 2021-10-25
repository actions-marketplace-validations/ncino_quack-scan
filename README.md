# Quack 🦆

![Rubber Duck](giant-rubber-duck.jpg?raw=true "Quack")

# Description

Run static code dependency scan using Black Duck Synopsys scan for NPM and PIP using Synopsys 6.0. For details on properties and the Black Duck Synopsis scan refer to their documentation here: https://blackducksoftware.github.io/synopsys-detect/6.0.0/. Inputs prefaced with bd are referring to arguments in the Black Duck scan. Descriptions are shared from the current documentation published.'

# Example Local Actions

Example local actions are kept in the example-local-actions directory. To use quack-scan checkout the project repository, run project setup, and install the java jdk before executing the scan.

example-local-action-0.yaml

```
name: quack-scan

on:
  release:
    types:
      - created

  # Allows you to run this workflow manually from the Actions tab. Only works after being merged into main branch.
  workflow_dispatch:

jobs:
  quack-scan:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        # Most builds are run on node 12. Can be run on 14 in the future.
        node-version: [ 12.x ]
        # Did not include Java 15 due to limitations in Synopsys scan.
        # It fails out when attempting to build and needs to be updated by BD.
        java-version: [ 13.x ]
        # Most builds are running 3.8 as of 3-16-2021.
        python-version: [ 3.8.x ]

    # Steps represent a sequence of tasks that will be ed as part of the job
    steps:
      # Checks-out the current repository under $GITHUB_WORKSPACE, so this job can access it.
      - uses: actions/checkout@v2

      # Sets up ssh agent and saves an access key.  Used in project setup for python and node.js.
      # Python searches for the ssh key on the ssh agent and node.js looks for the key in ~/.ssh/id_rsa.
      - name: Setup SSH Keys and Known Hosts
        env:
          SSH_AUTH_SOCK: /tmp/ssh_agent.sock
        run: |
          ssh-agent -a $SSH_AUTH_SOCK > /dev/null
          ssh-add - <<< "${{ secrets.GH_ACTIONS_PRIVATE_SSH_KEY }}"
          mkdir -p ~/.ssh
          echo "${{ secrets.GH_ACTIONS_PRIVATE_SSH_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa

######################################
### Node.js Setup and Installation ###
######################################
      # Checks for the existence of a Package.json and sets NODE_INCLUDED to true if Node.js is present.
      # Also sets RUN_SCAN to true if Node.js is found.
      - name: Check for Existence of a Package.json
        run: |
          if test -f "package.json"; then
            echo "NODE_INCLUDED=true" >> $GITHUB_ENV
            echo "Package.json found, will set up Node.js."
            echo "RUN_SCAN=true" >> $GITHUB_ENV
            echo "Will run Black Duck Scan."
          else
            echo "NODE_INCLUDED=false" >> $GITHUB_ENV
            echo "Package.json does not exist, will skip Node.js setup."
            echo "RUN_SCAN=false" >> $GITHUB_ENV
            echo "Will not run Black Duck Scan."
          fi

      # Attempts to load npm nodes from cache.  Creates a cache is one is not found.
      - name: Cache Node Modules
        if: ${{ env.NODE_INCLUDED  == 'true' }}
        id: node-cache
        uses: actions/cache@v2
        with:
          path: node_modules
          key: node-modules-${{ hashFiles('package-lock.json') }}

      # Setup Node.js for each version in the strategy matrix. Used in Node.js Project Setup.
      - name: Use Node.js ${{ matrix.node-version }}
        if: ${{ env.NODE_INCLUDED == 'true' && steps.node-cache.outputs.cache-hit != 'true' }}
        uses: actions/setup-node@v2
        with:
          node-version: ${{ matrix.node-version }}

      # Runs project install to create a scannable version of the project.
      # The registry target is set here in case a repo does not have a .npmrc file that defines the registry.
      # Does not run if cache is successfully loaded or node.js is not found in the repo.
      - name: Node.js Project Setup
        if: ${{ env.NODE_INCLUDED == 'true' && steps.node-cache.outputs.cache-hit != 'true' }}
        run: |
          git config --global url.ssh://git@github.com/.insteadOf https://github.com/
          npm i -g sfdx-cli
          npm config set //npm.pkg.github.com/:_authToken=${{ secrets.DEVOPS_GITHUB_TOKEN }}
          npm config set ${{ secrets.NPM_REGISTRY }}
          npm install

#####################################
### Python Setup and Installation ###
#####################################
      # Checks for the existence of a Package.json and sets PYTHON_INCLUDED to true if Pipfile is present.
      # Also sets RUN_SCAN to true if Pipfile is found.
      - name: Check for Existence of a Pipfile
        run: |
          if test -f "Pipfile"; then
            echo "PYTHON_INCLUDED=true" >> $GITHUB_ENV
            echo "Pipfile found, will set up Python."
            echo "RUN_SCAN=true" >> $GITHUB_ENV
            echo "Will run Black Duck Scan."
          else
            echo "PYTHON_INCLUDED=false" >> $GITHUB_ENV
            echo "Pipfile does not exist, will skip Python setup."
            echo "RUN_SCAN=false" >> $GITHUB_ENV
            echo "Will not run Black Duck Scan."
          fi

      # Attempts to Python Dependencies from cache.  Creates a cache is one is not found.
      - uses: actions/cache@v2
        id: python-cache
        if: ${{ env.PYTHON_INCLUDED == 'true' }}
        with:
          path: ${{ env.pythonLocation }}
          key: ${{ env.pythonLocation }}-${{ hashFiles('setup.py') }}-${{ hashFiles('dev-requirements.txt') }}

      # Setup for Python for each version in the strategy matrix. Used in Python Project Setup.
      # Does not run if cache is successfully loaded or python is not found in the repo.
      - name: Use Python ${{ matrix.python-version }}
        if: ${{ env.PYTHON_INCLUDED == 'true' && steps.python-cache.outputs.cache-hit != 'true' }}
        uses: actions/setup-python@v2
        with:
          python-version: ${{ matrix.python-version }}

      # Runs project install to create a scannable version of the project.
      # Uses an ssh key created in Setup SSH Keys and Known Hosts step.
      - name: Python Project Setup
        if: ${{ env.PYTHON_INCLUDED == 'true' && steps.python-cache.outputs.cache-hit != 'true' }}
        run: |
          pip install pipenv
          pipenv install --dev
        env:
          SSH_AUTH_SOCK: /tmp/ssh_agent.sock

#####################################
### Black Duck Scan Setup and Run ###
#####################################
      # Sets up the Java JDK for each version defined in the strategy matrix. Java is necessary for Black Duck.
      - name: Use Java JDK ${{ matrix.java-version }}
        if: ${{ env.RUN_SCAN == 'true' }}
        uses: actions/setup-java@v1.4.3
        with:
          java-version: ${{ matrix.java-version }}

      # Run BD scan
      - name: quack-scan
        uses: ncino/quack-scan@v1
        with:
          bd_url: ${{ secrets.BLACKDUCK_URL }}
          bd_api_token: ${{ secrets.BLACKDUCK_API_TOKEN }}
          detect_detector_search_depth: "0"

      # If there was a failure in running post an alert to Slack.
      - name: Send Alert to Slack on a Failure
        if: ${{ failure() }}
        uses: 8398a7/action-slack@v3
        with:
          github_base_url: https://github.com/ # Specify your GHE
          status: ${{ job.status }}
          fields: repo,commit,author,action,eventName,workflow
          icon_url: 'https://raw.githubusercontent.com/ncino/quack-scan/release/giant-rubber-duck.jpg'
          channel: '#devops-alerts'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

# Inputs

- bd_url:

  - description: 'URL of the Black Duck server.'
  - required: true

- bd_api_token:

  - description: 'The API token used to authenticate with the Black Duck Server.'
  - required: true

- detect_project_name:

  - description: 'An override for the name to use for the Black Duck project.
    If not supplied, Detect will attempt to use the tools to figure
    out a reasonable project name. If that fails, the final part of
    the directory path where the inspection is taking place will be used.'
  - required: false
  - default: '${{ github.event.repository.name }}'

- detect_project_version_name:

  - description: 'An override for the version to use for the Black Duck project.
    If not supplied, Detect will attempt to use the tools to figure
    out a reasonable version name. If that fails, the current date
    will be used.'
  - required: false
  - default: '${GITHUB_REF##\*/}'

- detect_required_detector_types:

  - description: 'The set of required detectors. If you want one or more detectors
    to be required (must be found to apply), use this property to
    specify the set of required detectors. If this property is set,
    and one (or more) of the given detectors is not found to apply,
    Detect will fail.'
  - required: false
  - default: 'NPM'

- detect_included_detector_types:

  - description: 'By default, all tools will be included. If you want to include
    only specific tools, specify the ones to include here. Exclusion
    rules always win. If you want to limit Detect to a subset of its
    detectors, use this property to specify that subset.'
  - required: false
  - default: 'NPM,PIP'

- detect_tools:

  - description: 'The tools Detect should allow in a comma-separated list. Tools
    in this list (as long as they are not also in the excluded list)
    will be allowed to run if all criteria of the tool are met.
    Exclusion rules always win. This property and detect.tools.excluded
    provide control over which tools Detect runs.'
  - required: false
  - default: 'DETECTOR'

- detect_detector_search_depth:

  - description: 'Depth of subdirectories within the source directory to which
    Detect will search for files that indicate whether a detector
    applies. A value of 0 (the default) tells Detect not to search
    any subdirectories, a value of 1 tells Detect to search first-level
    subdirectories, etc.'
  - required: false
  - default: '15'

- detect_npm_include_dev_dependencies:
  description: 'Set this value to false if you would like to exclude your dev
  dependencies when ran.'

  - required: false
  - default: 'false'

- detect_detector_search_exclusion_paths:

  - description: 'A comma-separated list of directory paths to exclude from detector
    search. (E.g. foo/bar/biz will only exclude the biz directory
    if the parent directory structure is foo/bar/.) This property
    performs the same basic function as detect.detector.search.exclusion,
    but lets you be more specific.'
  - required: false
  - default: ''

- detect_detector_search_exclusion_defaults:

  - description: 'If true, the bom tool search will exclude the default directory names.
    See the detailed help for more information. If true, these directories
    will be excluded from the detector search: bin, build, .git, .gradle,
    node_modules, out, packages, target.'
  - required: false
  - default: 'true'

- detect_detector_search_exclusion_patterns:
  - description: 'A comma-separated list of directory name patterns to exclude from
    detector search. While searching the source directory to determine
    which detectors to run, subdirectories whose name match a pattern
    in this list will not be searched.'
  - required: false
  - default: ''
