# Codecademy

[![Code Climate](https://codeclimate.com/repos/5318c3b9e30ba02b7a00a5a2/badges/70da304dd7fb8842e535/gpa.svg)](https://codeclimate.com/repos/5318c3b9e30ba02b7a00a5a2/feed)

Environment | Branch | URL | Status
--- | --- | --- | ---
Staging | `master` | https://staging.codecademy.com | [![CircleCI](https://circleci.com/gh/codecademy-engineering/Codecademy/tree/master.svg?style=shield&circle-token=79d0879385ff207293cc57806c27c057b3181347)](https://circleci.com/gh/codecademy-engineering/Codecademy/tree/master)
Production | `production` | https://www.codecademy.com | [![CircleCI](https://circleci.com/gh/codecademy-engineering/Codecademy/tree/production.svg?style=shield&circle-token=79d0879385ff207293cc57806c27c057b3181347)](https://circleci.com/gh/codecademy-engineering/Codecademy/tree/production)

The code repository of Codecademy ✨

## Getting Started

Check out the [development environment setup page in Notion](https://www.notion.so/codecademy/Local-Dev-Setup-2-0-77aa185969fb47b1ab0b418c9343e079) for prerequisites.

You should also do the [Infrastructure On-Boarding](https://www.notion.so/codecademy/Infrastructure-Onboarding-9d33fad1b33b424aa608cee69ea22a94) to enable full functionality.

### Generating a Github Token

When running `bundle install`, bundler needs a github token to access codecademy-engineering private repositories.

Follow the github instructions for [Creating a Personal Access Token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line). Create a token for `repo` access.

Once this token is generated,

1. copy the value to your `~/.zshrc` or `~/.bashrc`:
2. `export BUNDLE_GITHUB__COM=<generatedToken>`
3. `source` this file
4. Enable SSO for this token

You are now ready to run `bundle install` in the monolith.

### Configure the Environment

1. Install Ruby dependencies:
    ```console
    bundle install
    ```
1. Generate the `.env` config file:
    ```console
    $ ./script/create_dotenv.sh
    ```

### Run the App

1. Start the dependent services (mongo, redis + memcached):
    ```sh
    make services
    ```
    > **NOTE** This is just a shortcut for `docker-compose up -d mongo redis memcached`
1. Run the rails server:
    ```sh
    bundle exec rails server
    ```
1. Run webpack:
    ```sh
    yarn run start
    ```
    > **Troubleshooting** If you receive errors about missing files when first running this, run `yarn run build:all` to generate them.
1. **Optional:** If you need background job processing, run sidekiq:
    ```sh
    bundle exec sidekiq -C config/sidekiq.yml
    ```
Once Codecademy is running, the web app should be available at http://localhost:3000/ 🙌

#### Optional: Run things in Docker

Any of the backend app components can also be run in Docker.

For an interactive prompt, run `make up`.
> **NOTE** you must run `make bake` FIRST if running `make up` for the very first time, or you are building the container from scrath. Not doing so will throw SSH key errors.

> **Troubleshooting** If you recieve an SSH key error or a error about `mount` not being supported, you probably need to run `make bake` first/again

Or run the commands yourself:
- Build the images: `make bake`
- Run Sidekiq: `docker-compose up -d sidekiq`
- OR run Rails + Sidekiq: `docker-compose up -d rails`

### Create a User

Visit [the web app](http://localhost:3000/) and create yourself a user.

To give the user admin access, use the rails console:
```sh
bundle exec rails console
> User.find_with_identifier("<your email here>").add_role!("admin")
```

### Cleanup

`make down` will shutdown all Docker containers (say yes to the `Clean?` option to remove associated volumes and images defined by the compose file). `CTRL+C` exits native processes.

## CI/CD and Deployment

### Zoo Servers

"Zoo" servers are testing servers where you can publish code from a Git branch that isn't ready for production yet.

To deploy to a zoo server:

1. Log into [envs](https://envs.codecademy.com) with the username and password you set up during dev onboarding for Nagios
1. Find a row marked as Unlocked (green)
1. Paste your branch name in that row, where it says _'Branch to deploy'_ and press <kbd>Enter</kbd>

The deployment process will start shortly after.
You'll receive a mention in the `#deploys` channel on Slack when it's ready.

### Integration Process

![Deployment Diagram](https://docs.google.com/drawings/d/e/2PACX-1vQ6gW2AjTXVbOre_j58KwG0PlDx1E6sLKN0mo_SEatZhKJjLRekq8LYz_g_B0-NSsTNiuxCsYYHKlrl/pub?w=1125&h=478)

1. Create a **Pull Request** from `feature-branch` to `master` and **ensure checks pass**
1. Check the #deploys channel in slack to see if there is a queue. If there is one, join it and wait patiently, and when it's your turn, continue to the next step. If there isn't a queue, you can continue to the next step immediately.
1. **Merge** - the commit in `master` will automatically trigger a CircleCI job that deploys to staging and locks the staging env
1. **Monitor** progress on [CircleCI][circleci-master]
1. **Verify** that staging works and watch [NewRelic][newrelic-staging] for errors
    - If you broke staging, your #1 priority is to fix it: see [Fix a Broken Build](#fix-a-broken-build)

At this point your code is **integrated**. It is merged into `master`, which teammates will be branching from, deployed on staging, and part of the "source of truth", the annals of git history.

#### Next Steps

- Either release your feature to production, see [Release to Production](#release-to-production)

> **NOTE:** If you are releasing your feature to production, **do not** UNLOCK staging on [envs][envs] before releasing to production.

- Or not. For instance, there is a code freeze, or the change is small and does not warrant its own production release. As part of `master`, it will de facto be bundled in the next release, so be cognizant of that when it happens.

> **NOTE:** Remember to UNLOCK staging on [envs][envs] if you are not releasing to production. It is courteous to post in `#engineering` on Slack that you are leaving a change on master.

### Release to Production

1. Create a [**Pull Request**](https://github.com/codecademy-engineering/Codecademy/compare/production...master) from `master` to `production` and **ensure checks pass**
1. **Merge** - the commit in `production` branch will automatically trigger a CircleCI job that deploys to production
1. **Monitor** progress on [CircleCI][circleci-production]
1. **Verify** that production works and watch [NewRelic][newrelic] for errors
    - If you broke production, see [Hotfixing Production](#hotfixing-production)
    - Otherwise, **unlock production and staging on [envs][envs] and you are done!**

> **NOTE:** Remember to UNLOCK production and staging on [envs][envs] when you are finished verifying.
>
> You must manually UNLOCK, as there is no automated way to know when you'll be done testing,
> however you DO NOT NEED TO LOCK [ENVS][envs] prior to merging/deploying,
> the CircleCI deploy job will lock envs automatically, which will prevent others from merging until you unlock.
> And in fact, you may lock YOURSELF from deploying because your ENVS username != your GitHub username.

### Fix a Broken Build

When master/staging is broken.

Aside from taking priority over other features and PRs, the process is exactly the same as [integrating](#integration-process) any other code:
1. **PR** your `bugfix-xyz` branch into `master`
1. **Merge**
1. **Monitor** progress on [CircleCI][circleci-master]
1. **Verify** that staging works

See [note on rollbacks](#note-on-rollbacks).

### Hotfixing Production

1. Checkout a hotfix branch from `master`. Fix the issue on that branch.
    ```
    git checkout master
    git pull
    git checkout -b hotfix-my-bug
    ```
1. PR and merge the hotfix branch into `master`. This will trigger a build and deploy to staging; verify the bugfix on staging
1. PR and merge the hotfix branch into `production`. This will trigger a build and deploy to production; verify the bugfix on production

> If your issue is extremely time sensitive, consider running the `production` merge at the same time as the `master` merge

### Manual Override

If CircleCI is down, or a server is borked, you can manually deploy:

```sh
bundle exec cap {production|staging} deploy -s revision={branch|tag|commit-sha}
```

### Note on Rollbacks

Prefer not to rollback unless it is an absolute necessity (and staging should never really fit that criteria). Rollbacks introduce a discrepancy in the process in that they temporarily remove the problem from the server, but the bug still exists in source control, which can lead to bug regression if not handled properly.

It's better to fix the issue properly in the source code and let the CICD pipeline deploy the new, fixed code to the servers.

In order of preference:
1. Best case: fix your bug properly and re-deploy new changes
1. Second best case: When you can't fix or don't have time but need to fix production, simply `git revert` the offending commit. E.g.:
    ```sh
    git checkout production
    git checkout -b hotfix-revert-my-buggy-code
    git revert <sha of offending commit>
    # Now follow the exact same Hotfixing process with `hotfix-revert-my-buggy-code`
    ```
1. Worst case: Rollback. As this is not typical or desired, it's not part of the CICD flow, i.e. it's not a button on ENVS or a configuration in circleCI, so you need to do it manually:
    ```sh
    bundle exec cap production deploy:rollback
    ```
    > **NOTE** after rolling back you would still need to fix your bug (or revert) in master and production! Otherwise the issue will return on next deployment

### Webpack Assets

Webpack assets are uploaded to S3 when any build is deployed to Zoo/Staging/Production. Production S3 webpack build assets are automatically replicated from nonprod S3 asset bucket. This process is eventually consistant, so generally the webpack build should exist in production S3 Bucket, but it may miss. This is not really an issue and will only force a local asset build, causing longer deploy time.
- Webpack S3 bucket in nonprod (zoo/staging): `builds.monolith.nonprod.us-east-1`
- Webpack S3 bucket in production: `builds.monolith.prod.us-east-1`

[envs]: https://envs.codecademy.com
[circleci-master]: https://circleci.com/gh/RyzacInc/workflows/Codecademy/tree/master
[circleci-production]: https://circleci.com/gh/RyzacInc/workflows/Codecademy/tree/production
[newrelic]: https://rpm.newrelic.com/accounts/307474/applications/2210607/filterable_errors#/table?top_facet=transactionUiName&barchart=barchart
[newrelic-staging]: https://rpm.newrelic.com/accounts/307474/applications/3656061/filterable_errors#/table?top_facet=transactionUiName&barchart=barchart

### Docker Images
The monolith builds docker images for use in other CI/CD pipelines. We put these images in the `nonprod` AWS account.

To build an image and push to ECR, run the following from the root of the repository:

```console
$ aws-vault exec nonprod -- make build -f docker/Makefile
$ aws-vault exec nonprod -- make push -f docker/Makefile
```

Follow the prompts to build an image from the available Dockerfiles in `./docker/`.

To build an image without prompts, specify the `IMAGE` and `DIST` like so:

```console
$ aws-vault exec nonprod -- make build -f docker/Makefile IMAGE=browsers DIST=ubuntu
```

#### NOTES
* `rails-base` should be built first. It is the base for `browsers` and `rails`. The same commit hash base is required for both images.

* In CircleCI, anytime anything is changed in the docker directory and comitted, it will build all the docker images. Currently these are intended as checks, and are not pushed to ECR on build.
## PR Environments in EKS (managed Kuberdoodle)
Once done with a branch and ready to open a Pull Request, you have the ability to create an environment while the PR is open. Here's how to create one:

1. Add the `preview` [GitHub label](https://help.github.com/en/github/managing-your-work-on-github/applying-labels-to-issues-and-pull-requests) to any new or open Pull Request
1. The [GitHub status check]() `Pull Request Preview / Deploy (pull_request)` will show the status of the GitHub Action responsible for deploying your PR preview environment
1. If successful, the status `Pull Request Preview URL` "Details" link will open your PR preview environment (click "[show all checks](https://help.github.com/en/github/administering-a-repository/about-required-status-checks)" if they have already completed).
1. To stop building your environment for a time (ex: if you don't want a build on every pushed batch of commits), remove the `preview` label from step 1.
1. Once a PR is merged or closed, a github action will automatically cleanup the environment.

The preview URL pattern is `pr-<NUMBER>.dev-eks.codecademy.com`

### Accessing PR Environment in EKS

The tools are all the same (kubectl, kubectx, kubens) as before.

Run `kubectx` to list available contextes
```
dev.k8s.codecademy.com
development-eks
k8s.prod.codecademy.com
staging.k8s.codecademy.com
```

If you dont have `development-eks` in your kubecontexes follow setup instructions [here](https://www.notion.so/codecademy/Infrastructure-Kubernetes-Tools-387ec8a0784146afb29f62e40342493e).

Make sure you have the eks cluster context set
```
kubectx development-eks
```

The biggest difference between EKS and the old clusters is that we now use AWS IAM authentication (instead of shared).
This means that you have to wrap any `kubectl`command with `aws-vault exec nonprod`

Examples:

To get all the pods running in your pr env (namespace: `pr-<NUMBER>`):
```
aws-vault exec nonprod -- kubectl get pods -n pr-<NUMBER> 
```

To exec into a pod running rails:

```
aws-vault exec nonprod -- kubectl exec -it -n pr-<NUMBER> <RAILS-POD-NAME> bash
```

From here you can run the rails console, look atfiles etc.

To follow logs:

```
aws-vault exec nonprod --  kubectl logs -n pr-<NUMBER> <rails-pod-name> -f
```
And as before logs would be in scalyr as well

If you get tired of typing `-n pr-<NUMBER>` you can set the namespace globally with
```
aws-vault exec nonprod -- kubens pr-<NUMBER>
```
## PR Environments

Once done with a branch and ready to open a Pull Request, you have the ability to create an environment while the PR is open. Here's how to create one:

1. Add the `preview` [GitHub label](https://help.github.com/en/github/managing-your-work-on-github/applying-labels-to-issues-and-pull-requests) to any new or open Pull Request
1. The [GitHub status check]() `Pull Request Preview / Deploy (pull_request)` will show the status of the GitHub Action responsible for deploying your PR preview environment
1. If successful, the status `Pull Request Preview URL` "Details" link will open your PR preview environment (click "[show all checks](https://help.github.com/en/github/administering-a-repository/about-required-status-checks)" if they have already completed).
1. To stop building your environment for a time (ex: if you don't want a build on every pushed batch of commits), remove the `preview` label from step 1.
1. Once a PR is merged or closed, a github action will automatically cleanup the environment.

The preview URL pattern is `pr-<NUMBER>.monolith.dev.codecademy.com`

### Accessing PR Environment

To exec into a running PR environment, you will need access to the kubernetes dev cluster.

Instructions on how to access the dev cluster can be found in this [notion doc](https://www.notion.so/codecademy/Infrastructure-Kubernetes-Tools-387ec8a0784146afb29f62e40342493e)

Once done, you can use `kubectl` to access your PR environment, much like our services.

PR environments are located in the following namespace:

```
pr-<prNumber>
```

To exec (ssh) into your running rails container, run the following:

```
$ kubectl get pods -n pr-<prNumber> # get rails pod name
$ kubectl exec -it -n pr-<prNumber> <rails-pod-name> bash
```

From here you can run the rails console, look at files, etc.

All PR environment logs are located in Scalyr. You can also access logs via `kubectl`:

```
$ kubectl get pods -n pr-<prNumber> # get rails pod name
$ kubectl logs -n pr-<prNumber> <rails-pod-name> -f
```

## Infrastructure & Services Notes
The infrastructure is managed by [Terraform](https://www.terraform.io/)

[Code](https://github.com/codecademy-engineering/infrastructure)

### Appplication Webapp

### Sidekiq Workers

Zoo and Staging use localhost sidekiq process
The default configuration is located at `config/sidekiq.yml`

Production has dedicated worker hosts. These run/are deployed with the exact same code as the webapps but do not run the rails application and do not serve requests, only a sidekiq process.

- `worker.monolith.production.us-east-1.asg`
- `worker.monolith.production-reports.us-east-1.asg`

There should only be a single instance of the reports worker as it runs cron jobs that should not be duplicated. The non reports workers can be scaled up arbitrarily

Sidekiq configuration and the associated `god` process/configs in `config/god/*` and `config/sidekiq/*.yml` is deprecated in favor of [systemd](https://github.com/codecademy-engineering/infrastructure/tree/develop/salt/states/codecademy/files/monolith) managed services

systemd manages a parent process that is linked to 2 child processes:
- `sidekiq-metaevents`
- `sidekiq-default`

Regardless of the contents of `config/sidekiq.yml`, the `sidekiq-metaevents` service will ALWAYS watch the metaevents queue, there is no need to define the metavents worker. The `sidekiq-defaults` service will read `config/sidekiq.yml` and act based on its contents.

Read documentation before creating new queues
- [Official Configuration Docs](https://github.com/mperham/sidekiq/wiki/Advanced-Options)
- [Extra Documentation](https://www.notion.so/codecademy/sidekiq-9ce3acc42dac4961895185a5a680c756)

Note: Non-God run Sidekiq processes may need seperate process if testing concurrency. Talk to I&S Team if you have conucurrency concerns

#### Monitoring
Infrastructure is monitored mainly though [NewRelic Infrastructure](https://infrastructure.newrelic.com/accounts/307474/hosts/system)

### Sidekiq
Outside of loading the localhost sidekiq web gui, we expose a [sidekiq-admin](https://github.com/codecademy-engineering/sidekiq-admin) webpage that allows monitoring/administrative actions as a k8s service. Credentials are located in 1Pass

- [Production Sidekiq Admin](https://sidekiq-admin.svc.prod.codecademy.com/)
- [Staging Sidekiq Admin](https://sidekiq-admin.svc.staging.codecademy.com/)

### nginx
We use [nginx](https://en.wikipedia.org/wiki/Nginx) as a reverse proxy in front of [unicorn](https://bogomips.org/unicorn/)
- [Production New Relic Monitoring](https://infrastructure.newrelic.com/accounts/307474/integrations/onHostIntegrations/accounts/2/nginx/dashboard?filters=%7B%22and%22%3A%5B%7B%22is%22%3A%7B%22label.NETWORK_NAMESPACE%22%3A%22production%22%7D%7D%5D%7D)

Configuration is controlled by the [I&S Team](https://github.com/codecademy-engineering/infrastructure/blob/develop/salt/states/nginx/files/nginx-monolith.conf) and can be found in `/etc/nginx/nginx.conf`

### Redis
Redis is managed by a HA Elasticache Redis Cluster (Not Running Cluster Mode)
In dev/staging there is a single Elasticache Redis cluster.

In production, there are 3 clusters:
- cohort: Manages feature flags
- cache: Cache
- persistent: Persistent data

The URL for the clusters (and other configuration variables) can be found in `/var/codecademy/env`
If you require CLI access to the cluster via `redis-cli`, You must initiate the connection from a webapp or worker. There is no ability to directly connect to it from the internet
- [Production New Relic Monitoring](https://infrastructure.newrelic.com/accounts/307474/integrations/aws/accounts/22485/elasticache/dashboard)
- [Nonprod New Relic Monitoring](https://infrastructure.newrelic.com/accounts/307474/integrations/aws/accounts/22484/elasticache/dashboard)

### Memcached
Memached loads its configuration via the MEMCACHE_SERVERS ENV Variable [Docs](https://github.com/petergoldstein/dalli#usage-with-rails-3x-and-4x)

Memcached is managed by a HA Elasticache Memcached Cluster. The URL can be found in `/var/codecademy/env`
- [Production New Relic Monitoring](https://infrastructure.newrelic.com/accounts/307474/integrations/aws/accounts/22485/elasticache_memcached/dashboard?filters=%7B%22and%22%3A%5B%7B%22is%22%3A%7B%22provider.cacheClusterId%22%3A%22mono-cache-production%22%7D%7D%5D%7D)
- [Nonprod New Relic Monitoring](https://infrastructure.newrelic.com/accounts/307474/integrations/aws/accounts/22484/elasticache_memcached/dashboard)

### Logs
We use the service [scalyr](https://www.scalyr.com/) to aggregate logs
- [Production Webapp/Worker Logs](https://www.scalyr.com/events?filter=($server_group%20%3D%3D%20%22monolith-worker%22%20%7C%7C%20$server_group%20%3D%3D%20%22monolith-webapp%22)%20$network_namespace%20%3D%3D%20%22production%22&mode=log&teamToken=0eMeCHpqDg9OzbKH%2FIt8Fw--&startTime=10%20min)

Logfiles can be found in `/var/log/codecademy` which is a symlink to `/app/codecademy/current/log/`

### Shell Access
We use an AWS Service to access shell on hosts. [Instructions to use](https://www.notion.so/codecademy/Infrastructure-Onboarding-9d33fad1b33b424aa608cee69ea22a94#e60acea0cbd948d48084e4f842a883b6)

### Misc. Infra Notes
The application hosts are intended to be treated as [cattle](https://devops.stackexchange.com/questions/653/what-is-the-definition-of-cattle-not-pets). They are deployed as an autoscaling group, and so any instance that is terminated should automatically be recreated and boot up with a functioning application running and ready to serve requests.
