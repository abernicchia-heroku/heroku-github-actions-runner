## GitHub Actions Heroku-hosted Docker Runner

This project defines a `Dockerfile` to run a [self-hosted](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners) Github Actions runner.

The runner is hosted on [Heroku as a docker image](https://devcenter.heroku.com/articles/build-docker-images-heroku-yml) via `heroku.yml`.

The `start.sh` script, inspired by Michael Herman's [tutorial](https://testdriven.io/blog/github-actions-docker/),
auto registers the newly spun up Heroku dyno as a runner for a GitHub organization.

## Quick Start

**Things you'll need**

- Administrator access to your GitHub organization
- Administrator access to your Heroku organization
- GitHub personal access token with **admin:org** and **repo** scopes
- Heroku API token from a non-SSO user (service/automation user) with **view** and **deploy** access
- [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli)
- [Git CLI](https://git-scm.com/)

The setup requires configurations in both your GitHub organization and your Heroku organization.

You will switch between them throughout the following instructions.

1. In GitHub, enable GitHub Actions for your organization
    - https://github.com/organizations/{YOUR_ORGANIZATION}/settings/actions
    - Under **Policies**
        - Choose **Allow {YOUR_ORGANIZATION}, and select non-{YOUR_ORGANIZATION}, actions and reusable workflows**
        - Select **Allow actions created by GitHub**
        - Click **Save**

2. In GitHub, add your Heroku private space's IP addresses to your organization's allow list (see this [article](https://docs.github.com/en/enterprise-cloud@latest/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/managing-allowed-ip-addresses-for-your-organization#adding-an-allowed-ip-address)) and check the **Enable IP allow list** box

3. In GitHub, create a personal access token with **admin:org** and **repo** scopes (see this [article](https://docs.github.com/en/enterprise-server@3.9/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-personal-access-token))
    > Don't forget to authorize your access token to SSO to your organization

4. In Heroku, create a new app in your private space

5. In Heroku, add two configuration variables to the new app
    - `GITHUB_ACCESS_TOKEN` with the token you created previously
    - `GITHUB_ORGANIZATION` with the name of your organization

6. From the Heroku CLI login as the service/automation Heroku user to create an API token (SSO users cannot create tokens)

7. Generate a new Heroku API key
    - `heroku authorizations:create -d "GitHub self-hosted actions automation" --expires-in=<set expiration time in seconds>` setting an adequate expiration time

8. Login as the Heroku administrator and grant the service/automation Heroku user access to **view** and **deploy** to your new app
    - https://dashboard.heroku.com/apps/{YOUR_APP}/access

9. In GitHub, add three organization secrets to store the Heroku information
    - https://github.com/organizations/{YOUR_ORGANIZATION}/settings/secrets/actions
    - `HEROKU_API_KEY` with the api key you created previously
    - `HEROKU_API_USER` with the username who owns the api key
    - `HEROKU_ACTIONS_RUNNER_APP` with the name of your new Heroku app

10. Locally, clone and deploy this repository to your Heroku app

    ```shell
    git clone https://github.com/abernicchia-heroku/heroku-github-actions-runner.git
    heroku git:remote --app YOUR_HEROKU_APP
    heroku apps:stacks:set --app YOUR_HEROKU_APP container
    git push heroku main
    ```

11. In Heroku, scale your **runner** resource appropriate for your expected usage
    - https://dashboard.heroku.com/apps/{YOUR_APP}/resources
    - A single dyno can run one GitHub Actions job at a time
    - Recommended: Private-M dyno type scaled to 4 dynos

Voila!

Now when GitHub Action workflows are launched by your repositories, GitHub will orchestrate
with your Heroku-hosted runner to do the work just as if you were using GitHub-hosted runners.

## Keeping Your Runner Updated

GitHub frequently releases updates to the GitHub Action runner package.

If you don't keep the package up-to-date then GitHub won't enqueue jobs.

This project includes a workflow that can be run manually or once a week. It will rebuild the docker container
and download the latest updates automatically and it will deploy automatically to your Heroku self-hosted runner app.

To take advantage of this automation you need to [fork](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/fork-a-repo) or [mirror](https://docs.github.com/en/repositories/creating-and-managing-repositories/duplicating-a-repository#mirroring-a-repository-in-another-location) this repository to your private organisation's repository and enable workflows run.

## GitHub Runner Script Usage

The following is how to use the `config.sh` and `run.sh` scripts installed by the runner package (see https://github.com/actions/runner/blob/main/src/Runner.Listener/Runner.cs).

```
Commands:
 ./config.sh         Configures the runner
 ./config.sh remove  Unconfigures the runner
 ./run.sh            Runs the runner interactively. Does not require any options.

Options:
 --help     Prints the help for each command
 --version  Prints the runner version
 --commit   Prints the runner commit
 --check    Check the runner's network connectivity with GitHub server

Config Options:
 --unattended           Disable interactive prompts for missing arguments. Defaults will be used for missing options
 --url string           Repository to add the runner to. Required if unattended
 --token string         Registration token. Required if unattended
 --name string          Name of the runner to configure (default hostname on Linux - from C# Environment.MachineName)
 --runnergroup string   Name of the runner group to add this runner to (defaults to the default runner group)
 --labels string        Custom labels that will be added to the runner. This option is mandatory if --no-default-labels is used.
 --no-default-labels    Disables adding the default labels: e.g. 'self-hosted,Linux,X64'
 --local                Removes the runner config files from your local machine. Used as an option to the remove command
 --work string          Relative runner work directory (default _work)
 --replace              Replace any existing runner with the same name (default false)
 --pat                  GitHub personal access token with repo scope. Used for checking network connectivity when executing `./run.sh --check`
 --disableupdate        Disable self-hosted runner automatic update to the latest released version`
 --ephemeral            Configure the runner to only take one job and then let the service un-configure the runner after the job finishes (default false);

Examples:
 Check GitHub server network connectivity:
  ./run.sh --check --url <url> --pat <pat>

 Configure a runner non-interactively:
  ./config.sh --unattended --url <url> --token <token>

 Configure a runner non-interactively, replacing any existing runner with the same name:
  ./config.sh --unattended --url <url> --token <token> --replace [--name <name>]

 Configure a runner non-interactively with three extra labels:
  ./config.sh --unattended --url <url> --token <token> --labels L1,L2,L3;
```
## Credits and Technical Notes
Credits to the [owner](https://github.com/douglascayers/heroku-github-actions-runner) of the original project that inspired this updated version that:
- uses a new [GitHub Action](abernicchia-heroku/heroku-sources-endpoint-deploy-action) to build and deploy the runner on Heroku using the [sources endpoint API](https://devcenter.heroku.com/articles/build-and-release-using-the-api#sources-endpoint). This action allows you to deploy code living on GitHub repositories, even private, to apps running on Heroku without requiring the [Heroku GitHub integration](https://devcenter.heroku.com/articles/github-integration)
- uses ephemeral containers to allow [autoscaling](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/autoscaling-with-self-hosted-runners#using-ephemeral-runners-for-autoscaling) and [hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#hardening-for-self-hosted-runners) of self-hosted runners. Ephemeral runners are short-lived containers that are executed only once for a single job, providing isolated environments to reduce the risk of data leakage
- reduces the Docker image footprint
- includes all the recent GitHub self-hosted runners features
 