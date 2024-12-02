# TUF-on-CI Repository Initialization and Maintenance Manual

This page documents the initial setup of a TUF-on-CI repository as well as the
ongoing maintenance.

## New Repository Setup

1. [Create new repository](https://github.com/new?template_name=tuf-on-ci-template&template_owner=theupdateframework)
   using the tuf-on-ci template: the created repository contains all the required workflows.
1. Configure the new repository:
   * set _Settings->Pages->Source_ to `GitHub Actions`
   * Change _Settings->Environments->github-pages_ deployment branch from `main` to
     `publish`
   * Check _Settings->Actions->General->Allow GitHub Actions to create and approve pull requests_
     (not required if you are using a custom token, see below)
1. Clone the repository locally and [configure your local signing tool](SIGNER-SETUP.md).
1. Choose your online signing method and [configure it](ONLINE-SIGNING-SETUP.md):
   * Google Cloud KMS, Azure Key Vault, and AWS KMS are fully supported
   * Sigstore requires no configuration (but is experimental)
1. Run `tuf-on-ci-delegate sign/init` to configure the repository and to start the
   first signing event
   * The command creates a remote branch sign/init and the process for creating a new TUF-on-CI repository starts.
   * The roles signing method and expiring dates can be configured.
1. Sign the `root` and `targets` roles.
1. Push the changes to the remote repository and the "Signing event: sign/init" pull request will be created.
1. After the "TUF-ON-CI signing event" workflow that verifies that the signing event is successful, merge the pull request.
1. When this initial signing event branch is merged, the repository generates the first snapshot and timestamp, and publishes the first repository version.

It should be mentioned that the initial `tuf-on-ci-delegate` command does not look into `/targets` and does not include the hashes for the artifacts. For adding a target file(s), the procedure outlined below should be followed.

## Adding and modifying artifacts in the repository

Artifacts are stored in git (in the `targets/` directory) and are modified using normal git tools: the signing tool is not used. Artifact modification commits should get pushed to a branch on the repository (with a branch name starting with "sign/"): this creates a signing event for the artifact change allowing signers to sign that change.

Considering this, the following steps should be followed to add or modify an artifact. For simplicity, this example will use Sigstore for signing. Repository initialization mentioned before should have been done.

1. Update the local repository with the latest changes from the remote repository without modifying the working directory. This can be done with `git fetch`.
2. Create a new branch that starts with "sign/" and set its starting point to the latest commit from `origin/main`. This can be performed with `git fetch && git switch -c sign/add-a-target origin/main`.

3. Create `targets` directory. This can be done with `mkdir targets`.

4. Perform the desired modifications to the `targets` directory. For this example, a new target file named "file1.txt" will be added to the `targets` directory echo `"artifact" > targets/file1.txtv`.

5. Stage the changes `git add targets/file1.txt`.

6. Commit the changes with `git commit -m "New artifact file1.txt, managed by targets"`.

7. Push the branch with `git push origin sign/add-a-target`.

8. A TUF-on-CI signing event workflow is created. When it finishes, in the TUF-ON-CI signing event summary, it can be appreciated that the current signing event state tells that the "targets" role is still unsigned.

9. Signers can change the changes by running the `tuf-on-ci-sign sign/add-a-target`.
  
10. Authenticate with Sigstore for signing. In the case of using Yubikey, sign with it.

11. Press enter to push changes to `origin/sign/add-a-target`.

12. When pushing the changes, a TUF-on-CI signing event is created. If the signing procedure has been performed correctly, the current signing event state will show that the "targets" role has been verified and signed correctly.

13. Merge the pull request. When merging, the online-sign workflow is enabled, which is in charge of signing the `snapshot`and `timestamp`roles.

14. After the online-signing workflow, the publish worflow is performed.

It should be outlined that the online signing will be performed automatically and will generate new versions of `targets` and `timestamp`. The concurrence of this signing will depend on the expiring time chosen. However, when the `targets` or `root`metadata is expired, a new signing event will be created that will require the sign of the signers/maintainers. More information about signing can be found in [here](SIGNER-MANUAL.md).

## Modifying roles and creating new ones

Modifying a role is needed when:

* A new delegated role is created
* A new signer is invited to a role
* A signer is removed from a role
* The required threshold of signatures is changed

Roles are modified with `tuf-on-ci-delegate <event> <role>`.

* The event name can be chosen freely (and will be used as a branch name). If the signing
  event does not exist yet, it will be created as a result.
* The tool will prompt for new signers and other details, and then prompt to push changes
  to the repository.
* The push triggers creation of a signing event pull request. The repository will report the
  status of the signing event in the pull request and will notify signers there.

### Examples

TODO: Example: Creating a new delegated role

TODO: Example: Removing a signer

<details>
<summary>Example: Inviting a new root signer</summary>
In this example the root signers list contains a single signer, but it is modified to contain
two signers instead. The process is:

* tuf-on-ci-delegate is used to modify signers
* the new signer accepts the invitation and adds their keys to the delegating role's metadata
* the signers of the delegating role must accept the new key by signing the new
  version of delegating metadata

```shell
$ tuf-on-ci-delegate sign/add-fakeuser-2 root

Remote branch not found: branching off from main
Modifying delegation for root

Configuring role root
1. Configure signers: [@-fakeuser-1], requiring 1 signatures
2. Configure expiry: Role expires in 365 days, re-signing starts 60 days before expiry
Please choose an option or press enter to continue: 1
Please enter list of root signers [@-fakeuser-1]: @-fakeuser-1,@-fakeuser-2
Please enter root threshold [1]:
1. Configure signers: [@-fakeuser-1, @-fakeuser-2], requiring 1 signatures
2. Configure expiry: Role expires in 365 days, re-signing starts 60 days before expiry
Please choose an option or press enter to continue:
...
```

Once finished the changes are pushed to the signing event branch
which in the above example is `sign/add-fakueuser-2`.

The repository automation runs the [signing
automation](https://github.com/theupdateframework/tuf-on-ci-template/blob/main/.github/workflows/signing-event.yml)
that creates PRs with comments documenting current signing event state
and tags each signer. These comments (along with the PR commits) should
provide signers with a clear view of what is happening in the signing
event.

To accept the invitation and become a signer, the invitee runs
`tuf-on-ci-sign <event-name>` and provides information on what key to
use.

After this the delegating role signers (in this case root signers) accept
the new key by signing the delegating metadata version.
</details>

## Configuration and modifying workflows

tuf-on-ci workflows (with the exception of `publish`) are written in a way to minimize
need to modify the workflows: It may be useful to consider the workflows part of the
tuf-on-ci application. The intention with this is to make workflow upgrades easier:
tuf-on-ci release notes will mention when workflows change and typically the suggested
upgrade mechanism is to copy the modified workflows from tuf-on-ci-template.

Supported ways to configure and modify tuf-on-ci workflows:

* online signing is configured using signing method specific _Repository Variables_,
  see [ONLINE-SIGNING-SETUP.md](ONLINE-SIGNING-SETUP.md) for details
* A custom GitHub token can be optionally configured with _Repository Secret_
  `TUF_ON_CI_TOKEN`, see details below
* Workflow failure messages can be configured with `.github/TUF_ON_CI_TEMPLATE/failure.md`:
  Contents of this file will be included in issues that are opened if workflows fail. This is
  useful to e.g. notify the maintenance team with @-mentions.
* Signing pull request templates can be configured with
  `.github/PULL_REQUEST_TEMPLATE/signing_event.md`. Contents of this file will be included in
  the pull request message when non-maintainer signers contribute to signing events. This is
  useful to e.g. notify the maintenance team with @-mentions.
* The `publish` workflow can be customized to publish to a destination that is not
  the default GitHub Pages

### Custom GitHub token

tuf-on-ci uses GITHUB_TOKEN by default but supports using a custom fine-grained Github
token. This allows the project to limit the default GITHUB_TOKEN permissions
(in practice this means other workflows in the repository can operate with this lower
permission default token while tuf-on-ci workflows still have higher permissions).

The custom token needs the following repository permissions:

* `Actions: write` to dispatch other workflows when needed
* `Contents: write` to create online signing commits, and to create targets metadata
  change commits in signing event
* `Issues: write` to create issues on workflow failures
* `Pull requests: write` to create and modify signing event pull requests

To use a custom token, define a _repository secret_ `TUF_ON_CI_TOKEN` with a fine grained
token as the secrets value. No workflow changes are needed. Note that all automated comments
in signing event pull requests will be seemingly made by the account that created the custom
token: Creating the token on a "bot" account is sensible for this reason.

When a custom token is used, some repository security settings can be tightened:

* _Settings->Actions->General->Allow GitHub Actions to create and approve pull requests_
  can be disabled
* Custom token owner (bot) can be added to _Allow specified actors to bypass required
  pull requests_ list in GitHub branch protection settings, and _Settings->Branches->
  main->Require a pull request before merging_ can then be enabled
