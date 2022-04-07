# Retype git-ftp Action

This GitHub Action can publish a website built by [Retype](https://retype.com/) in a previous [`action-build`](https://github.com/retypeapp/action-build) step to a FTP server using the [`git-ftp`](https://github.com/git-ftp/git-ftp) tool.

## Introduction

The Retype's git-ftp action will commit and push back a Retype website to its GitHub repository, then sync it with the specified FTP host. The website needs to be built during a previous step by using the [retypeapp/action-build](https://github.com/retypeapp/action-build) action.

FTP credentials are required via the `ftp-host`, `ftp-pass`, `ftp-root` and `ftp-user` arguments in order to publish files to the FTP endpoint. Please use at least `ftp-pass` [as a secret](https://docs.github.com/en/actions/reference/encrypted-secrets#using-encrypted-secrets-in-a-workflow) in your repository.

Like the [action-github-pages](https://github.com/retypeapp/action-github-pages), this one includes options to push **to a branch**, a **directory within a branch**, and **create a pull request**, which can then be merged after human review.

Keeping a branch to hold change history allows for much faster FTP synchronization, as it uses git history to decide which files should be sent or removed on the remote server.

The following functionality is configurable by this action:

1. Target a `branch`, or
2. Target a `directory` within a branch
3. Configure whether the branch should be updated or copied (`update-branch`)
4. Configure GitHub API access token to allow the action to make a Pull Request when needed (`github-token`)

The following arguments are required for this action to run:

1. FTP hostname/IP (`ftp-host`)
2. FTP directory (root in the remote host, `ftp-root`)
3. FTP username (`ftp-user`)
4. FTP password (`ftp-pass` **make sure this [is passed as a secret](https://docs.github.com/en/actions/reference/encrypted-secrets#using-encrypted-secrets-in-a-workflow)!**)

### `.git-ftp` control file

The `git-ftp` tool uses a control file that is stored in the FTP server as **.git-ftp.log** additionally to the actual documentation website. See more in [the git-ftp documentation page](https://github.com/git-ftp/git-ftp/blob/master/man/git-ftp.1.md#description).

## Prerequisites

This action requires the output of the [retypeapp/action-build](https://github.com/retypeapp/action-build) in a previous step of the workflow.

## Usage

The following `retype.yaml` file demonstrates a typical scenario where a workflow should publish Retype website when changes are pushed to the repo. The fresh Retype powered website is then pushed to a `retype` branch which will hold change history to optimally update the remote FTP endpoint as commits are pushed on.

```yaml
name: Publish Retype powered website to FTP
on:
  workflow_dispatch:
  push:
    branches:
      - main

jobs:
  publish:
    name: Publish to the FTP host

    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - uses: retypeapp/action-build@latest

      - uses: retypeapp/action-git-ftp@latest
        with:
          ftp-host: ${{ secrets.FTP_SERVER_ADDRESS }}
          ftp-root: public_html
          ftp-user: ${{ secrets.FTP_USERNAME }}
          ftp-pass: ${{ secrets.FTP_PASSWORD }}
          branch: retype
          update-branch: true
```

## Inputs

It is recommended to pass all input prefixed with `ftp-` as [GitHub Encrypted Secrets](https://docs.github.com/en/actions/reference/encrypted-secrets) to preserve sensitive information. The output of `git-ftp` always prints the used credentials and addresses; if they are used as secrets, GitHub would replace them with `***`.

### `branch`

Specifies the target branch where the Retype output will be merged to.

- **Default:** `retype`
- **Accepts:** A string or the `HEAD` keyword. Examples: `gh-pages`, `main`, `website`, `HEAD`

#### Remarks

- **If the branch does not exist**: The action will create a new, orphan branch; then copy over the files, commit, and push.

- **If the branch exists:**
  - **And `update-branch` input is not `true`:** The action will fork from that branch into a new uniquely-named branch. The action will then wipe clean the entire branch (or a subdirectory within that branch, see `directory` below), then copy over the Retype output files, commit and push. This branch can then be merged or be used to make a pull request to the target branch. This action can create the pull request, see `github-token` input below.
  - **The `update-branch` input is `true`:** The action will wipe clean the branch (or directory), then copy over the Retype output, commit, and then push to the target branch.

- **The argument is `HEAD` keyword:** `update-branch` is implied `true` and Retype output files will be committed to the current branch. In this scenario, the action will ONLY run if a `directory` has been configured, as it would otherwise result in the replacement of all branch contents with the Retype output.

- **Detached HEAD:** If the argument is `HEAD` and commit SHA points to no branch or tag ([detached head](https://git-scm.com/docs/git-checkout#_detached_head)), then no commit will be possible and the action will fail.

- When wiping a branch or directory, if there is a `CNAME` file in its root, the existing `CNAME` file will be preserved.

- If the [`cname`](https://retype.com/configuration/project/) property is configured within your projects `retype.json` file, the `CNAME` file from the Retype build output will override it.

- If the `HEAD` keyword is used and `.`, `/`, or any path coinciding with the repository root is specified to `directory` input, then the whole branch data will be replaced with the generated documentation. Likewise, if the path passed as input conflicts with any existing path within the repository, it will be wiped clean by the action, replaced by the Retype output.

### `directory`

Specifies the root where to place the Retype output files. The path is relative to the repository.

- **Default:** null (root of the repository)
- **Accepts:** A string. Example: `docs`

#### Remarks

Using `/` or `.` is equivalent to not specifying an input, as it will be changing to the `.//` and `./.` directories respectively. Use of upper-level directories (`../`) is accepted but may result in files being copied outside the repository, in which case the action might fail due to being outside the repository boundaries.

### `ftp-host`

The hostname or IP address of the remote FTP server to sync files to. May specify the port with the format `<host_or_ip>:<port>`.

- **Default:** This argument is required and there's no default.
- **Accepts:** a string in the format `<hostname>` or `<hostname>:<port>` representing the remote FTP host.

### `ftp-pass`

The password used to authenticate with in the remote FTP host.

- **Default:** This argument is required and there's no default.
- **Accepts:** a string representing a password stored as a GitHub Secret.

#### Remarks

- Passwords beginning with `--` are not supported by `git-ftp`.
- **Use this as a [GitHub Encrypted Secret](https://docs.github.com/en/actions/reference/encrypted-secrets)**; failing to do so will expose the password in the workflow output log as git-ftp always prints it out.

### `ftp-root`

The root directory in the remote host where to place and sync the files.

- **Default:** This argument is required and there's no default.
- **Accepts:** a string representing a relative path to the FTP starting location.

#### Remarks

- use `/` or `.` if the files should be placed in the starting FTP directory.
- the directory **must exist** in the remote host; this may help avoid mistakes.
- common names for this are `public_html` for linux hosts and `wwwroot` for windows hosts.

### `ftp-user`

The username used to authenticate with in the remote FTP host.

- **Default:** This argument is required and there's no default.
- ** Accepts:** a string representing an username.

### `update-branch`

Indicates whether the action should update the target branch instead of creating a unique named branch.

- **Default:** null
- **Accepts:** `true` or `false`

#### Remarks

- When this option is configured, no pull request will be attempted even if `github-token` is specified.

- When `branch: HEAD` input is specified, this setting is assumed `true` as there will not be a reference to fork off (or pull request to) as `HEAD` is not a valid branch name.

- When this option is unset or `false` (default), although it may not push the changes to the target specified branch, it **will** sync with the remote FTP host. This means that, in case the branch or pull request is not accepted/merged, the next run will need a full sync, or fail. This can be fixed by merging the branch, or resetting the FTP directory.

### `github-token`

Specifies a GitHub Access Token that enables the action to make a pull request whenever needs to push a new branch. See [Using the GITHUB_TOKEN in a workflow](https://docs.github.com/en/actions/reference/authentication-in-a-workflow#using-the-github_token-in-a-workflow) and [Creating a personal access token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) for more details.

- **Default:** null
- **Accepts:** A string representing a valid GitHub Access Token either for User, Repository, or Action.

#### Remarks

- The action will never use the access token if
  1. the branch does not exist,
  2. `update-branch: true`, or
  3. `branch: HEAD` is specified.

## Examples

The following `retype.yaml` workflow file will serve as our starting template for most of the samples below.

```yaml
name: GitHub Action for Retype
on:
  workflow_dispatch:
  push:
    branches:
      - main

jobs:
  publish:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - uses: retypeapp/action-build@latest

      - uses: retypeapp/action-git-ftp@latest
        with:
          ftp-host: ${{ secrets.FTP_SERVER_ADDRESS }}
          ftp-root: public_html
          ftp-user: ${{ secrets.FTP_USERNAME }}
          ftp-pass: ${{ secrets.FTP_PASSWORD }}
          branch: retype
          update-branch: true
```

---

### Most common setup

```yaml
- uses: retypeapp/action-git-ftp@latest
  with:
    ftp-host: ${{ secrets.FTP_SERVER_ADDRESS }}
    ftp-root: public_html
    ftp-user: ${{ secrets.FTP_USERNAME }}
    ftp-pass: ${{ secrets.FTP_PASSWORD }}
    branch: retype
    update-branch: true
```

If the branch does not exist, it will be created as an orphan branch. This means the branch history won't be mixed with any other branch in the repository. This also means that, once the branch is deleted, GitHub will eventually wipe all history when it does its internal garbage collection.

---

### Publish to a subfolder in the FTP host

When the documentation should be held in a subdirectory in the target website, say, `/documentation`, this would be used:

```yaml
- uses: retypeapp/action-build@latest
  with:
    base: documentation

- uses: retypeapp/action-git-ftp@latest
  with:
    ftp-host: ${{ secrets.FTP_SERVER_ADDRESS }}
    ftp-root: public_html/documentation
    ftp-user: ${{ secrets.FTP_USERNAME }}
    ftp-pass: ${{ secrets.FTP_PASSWORD }}
    branch: ftp-history
    update-branch: true
```

**Notice:**
  - It is necessary to create the `public_html/documentation/` directory in the ftp site or the workflow will fail.

  - The `base` setting is required whenever the Retype website is not going to be hosted in the website's root. See more in [retype documentation on the `base` setting](https://retype.com/configuration/project/#base).

---

### Send to `main` branch, within the `docs` folder

This example will use a subfolder within the main branch to store history of the website changes.

```yaml
- uses: retypeapp/action-git-ftp@latest
  with:
    ftp-host: ${{ secrets.FTP_SERVER_ADDRESS }}
    ftp-root: wwwroot
    ftp-user: ${{ secrets.FTP_USERNAME }}
    ftp-pass: ${{ secrets.FTP_PASSWORD }}
    branch: main
    directory: docs
    update-branch: true
```

In this context, one would probably prefer the action to be triggered only on pushes/merges to the `main` branch, thus the workflow file would rather be configured like the following:

```yaml
name: GitHub Action for Retype
on:
  push:
    branches:
      - main
jobs:
  job1:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - uses: retypeapp/action-build@latest

      - uses: retypeapp/action-git-ftp@latest
        with:
          ftp-host: ${{ secrets.FTP_SERVER_ADDRESS }}
          ftp-root: wwwroot
          ftp-user: ${{ secrets.FTP_USERNAME }}
          ftp-pass: ${{ secrets.FTP_PASSWORD }}
          branch: main
          directory: docs
          update-branch: true
```

---

### Publish website on new release

In the following sample, whenever a new Release is created in GitHub, the documentation will be published to a FTP host, having its change history kept in the `retype` branch. If GitHub Pages was configured to the same branch, it should then become in sync with the FTP published one.

```yaml
name: Publish Retype powered website on a new release
on:
  push:
    branches:
      - main

jobs:
  publish:
    name: Assemble and publish docs
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - uses: retypeapp/action-build@latest

      - uses: retypeapp/action-git-ftp@latest
        with:
          ftp-host: ${{ secrets.FTP_SERVER_ADDRESS }}
          ftp-root: wwwroot
          ftp-user: ${{ secrets.FTP_USERNAME }}
          ftp-pass: ${{ secrets.FTP_PASSWORD }}
          update-branch: true
```

**Notice:** In case the GitHub Pages website needs a custom `base` setting, or the FTP host requires a different one, it's best to redo the [action-build](https://github.com/retypeapp/action-build) step with the right `base` for each publish target. The history for the builds should not be kept in the same branch+directory as the setting implies changes throughout all retype processed files. The next example shows how to deal with that.

See [Managing Releases in a Repository](https://docs.github.com/en/github/administering-a-repository/managing-releases-in-a-repository) for information about GitHub Releases.

---

### GitHub Pages and FTP simultaneously

This example shows how to handle publishing to different targets with different values for the `base` Retype setting. Here, the [action-github-pages](https://github.com/retypeapp/action-github-pages) will be used to publish to GitHub Pages default host (`organization.github.io/my-repo-name`) and [action-git-ftp](https://github.com/retypeapp/action-git-ftp) (this one) to publish to a FTP server's root public path.

As each target use different `base` values, different branch names are also required. This is because `base` changes refect throughout most built Retype files and it would lead `git-ftp` to re-upload more files than necessary.

```yaml
name: Publish on new release to bot GitHub Pages and FTP
on:
  release:
    types: [ published ]

jobs:
  publish:
    name: Assemble and publish docs
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Build documentation for GitHub Pages
        uses: retypeapp/action-build@latest
        with:
          base: my-repo-name

      - name: Publish built documentation to GitHub Pages
        uses: retypeapp/action-github-pages@latest
        with:
          branch: gh-pages

      - name: Switch back to original branch
        run: git checkout ${{ github.sha }}

      - name: Build documentation for FTP site
        uses: retypeapp/action-build@latest

      - name: Publish built documentation to FTP site
        uses: retypeapp/action-git-ftp@latest
        with:
          ftp-host: ${{ secrets.FTP_SERVER_ADDRESS }}
          ftp-root: public_html
          ftp-user: ${{ secrets.FTP_USERNAME }}
          ftp-pass: ${{ secrets.FTP_PASSWORD }}
          branch: ftp-site-history
          update-branch: true
```

Notice there's no `base` in the build step for the FTP site. This simply means we are going to use whatever is set in the `retype.json` file or use default, which is `/` (root).

Alternatively, from the example above, if we were to push to a FTP's `my-docs/` subdirectory, so that the documentation would be available at something like **http://www.mywebsite.com/my-docs/**, the build+publish steps would be changed to:

```yaml
- name: Build documentation for FTP site
  uses: retypeapp/action-build@latest
  with:
    base: my-docs

- name: Publish built documentation to FTP site
  uses: retypeapp/action-git-ftp@latest
  with:
    ftp-host: ${{ secrets.FTP_SERVER_ADDRESS }}
    ftp-root: public_html/my-docs
    ftp-user: ${{ secrets.FTP_USERNAME }}
    ftp-pass: ${{ secrets.FTP_PASSWORD }}
    branch: ftp-site-history
    update-branch: true
```

See [Managing Releases in a Repository](https://docs.github.com/en/github/administering-a-repository/managing-releases-in-a-repository) for information about GitHub Releases.

## See also

- [Retype documentation](https://retype.com/)
- Retype's [action-build](https://github.com/retypeapp/action-build)
- Retype's [action-github-pages](https://github.com/retypeapp/action-github-pages)
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [The git-ftp website]()