---
layout: post
title: "Automate deployments with GitHub Actions and Docker Swarm - CI/CD"
header: "Automate deployments with GitHub Actions and Docker Swarm - CI/CD"
slug: "automate-deployments-with-github-actions"
excerpt: "Once you can deploy an app to a remote Docker Swarm cluster from your local
machine, you might want to automate the process so that your app gets deployed every
time you push your code to your repository."
image: /assets/images/swarm-cicd/swarm-cicd.png
---

This is the second part of a series of posts on deploying a Rails app to a Docker Swarm
cluster on a remote VPS. In the first part, we set up the Swarm cluster on
Hetzner Cloud and deployed a Rails app to it from our local machine. If you haven't read
it yet, you can find it [here](/deploy-rails-docker-swarm-hetzner). In this post, we'll
automate the  deployment process with GitHub Actions, so that every time you push your
code to GitHub a CI/CD pipeline will build and deploy your app to the Swarm.

We are not going to cover linting and testing in this post, but if you want to add those
steps to your pipeline, you can check out this [PR](https://github.com/rails/rails/pull/50508) merged into Rails to add a workflow
that runs Brakeman, RuboCop, and the test suite on every push and pull request to the
main branch of the repository.

## Handle environment variables

Let's review the stack file we used to deploy the app to the Swarm in the first part of
this series.

```yaml
version: '3.7'

services:
  web:
    image: <dockerhub_username>/swarm-demo:prod
    depends_on:
      - database
    ports:
      - 80:3000
    environment:
      - RAILS_MASTER_KEY=c51c56d33c031224c4678f44dd7eafa0
      - POSTGRES_USER=swarm_demo
      - POSTGRES_PASSWORD=production-secure-password
    deploy:
      placement:
        constraints:
          - node.labels.type == app

  database:
    image: postgres:15
    environment:
      - POSTGRES_USER=swarm_demo
      - POSTGRES_PASSWORD=production-secure-password
    volumes:
      - db_data:/var/lib/postgresql/data
    ports:
      - 5432:5432
    deploy:
      placement:
        constraints:
          - node.labels.type == database

volumes:
  db_data:
```

We have some environment variables in the stack file that we don't want to expose in the
repository but definitely need them to deploy the app. In order to handle this, we will use the
`Bitwarden CLI` to fetch the secrets from Bitwarden and inject them into the stack file when
the CI/CD pipeline runs.

First, update the stack file so that it references an environment file instead of hardcoded values:

```yaml
version: '3.7'

services:
  web:
    image: <dockerhub_username>/swarm-demo:prod
    depends_on:
      - database
    ports:
      - 80:3000
    env_file:
      - .env
    deploy:
      placement:
        constraints:
          - node.labels.type == app

  database:
    image: postgres:15
    env_file:
      - .env
    volumes:
      - db_data:/var/lib/postgresql/data
    ports:
      - 5432:5432
    deploy:
      placement:
        constraints:
          - node.labels.type == database

volumes:
  db_data:
```

Second, create a free [Bitwarden account](https://vault.bitwarden.com/#/register?layout=default)
if you don't have one already and add `RAILS_MASTER_KEY`, `POSTGRES_USER`, and `POSTGRES_PASSWORD`
as `secure notes` to the default collection:

![Bitward note](/assets/images/swarm-cicd/bitwarden-note.png)

Now, install the Bitwarden CLI by following the instructions [here](https://bitwarden.com/help/cli/).

Once you have the Bitwarden CLI installed and authenticated, you can fetch the secured notes by running:

```bash
bw get notes <item-id>
```

### Hydrate `.env` file with Bitwarden secrets

We will implement a rake task to fetch the secrets from Bitwarden and inject them into the `.env` file.
This is a similar but simplified version of what [Kamal](https://kamal-deploy.org/docs/configuration)
does when you run `kamal envify`.

Create said rake task in `lib/tasks/env_variables.rake`:

```ruby
namespace :env_variables do
  # "Creates .env by evaluating .env.erb"
  task :hydrate do
    File.write(".env", ERB.new(File.read(".env.erb")).result, perm: 0o600)
  end
end
```

Now, create a `.env.erb` file in the root of your project with the following content:

```erb
<% if (session_token=`bw unlock --passwordenv BW_PASSWORD --raw`.strip) != "" %>
RAILS_MASTER_KEY=<%= `bw get notes <item-id> --session #{session_token}` %>
POSTGRES_USER=<%= `bw get notes <item-id> --session #{session_token}` %>
POSTGRES_PASSWORD=<%= `bw get notes <item-id> --session #{session_token}` %>
<% else raise ArgumentError, "session_token token missing" end %>
```

With this setup, when you run `rake env_variables:hydrate` the `.env` file will be created
as a result of `ERB.new(File.read(".env.erb")).result` executing the embedded Ruby code in the
`.env.erb` file.

Now that we can dynamically inject the secrets into the `.env` file, let's handle the CI/CD pipeline.

## GitHub Actions

GitHub Actions is a continuous integration and continuous delivery (CI/CD) platform
that allows you to automate your build, test, and deployment pipeline. GA uses the
concept of a **Workflow** which is defined by a YAML file in the `.github/workflows` directory.

We are going to configure a GitHub Actions workflow to be triggered when we push our changes
to the repository, deploying the app to the VPS using Docker Swarm.

First, let's create the `.github/workflows` directory and create a new file (workflow).
Let's call it `deploy.yml` and add the following code. We will break down the code right after.

{% gist ce72a7dd6c2cec8beba1cf1d25717ad1 %}

Let's break down the `name`, `run-name`, and `on` keys:

```yaml
name: Deploy
run-name: Deploying to Swarm cluster
on:
  push:
    branches: [main]
```

The `name` key is the name of the workflow and the `run-name` key is the name for workflow
runs, which will appear in the list of workflow runs in the GitHub Actions tab. The `on` key
defines the events that trigger the workflow. In this case, the workflow will run when we push
to the `main` branch.

The `jobs` key defines the jobs that will run in the workflow. In this case, we have only one
job called `deploy` that runs on an `ubuntu-latest` runner.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      ...
```

The `steps` key defines the steps that will run in the job. Each step is a task that can run
commands or actions. The **first** step checks out the codebase, the **second** step sets up Ruby on the runner,
the **third** step sets up Node.js so we can use the Bitwarden CLI, which is installed in the **fourth** step to
hydrate the `.env` file. The **fifth** and **sixth** steps set up Docker Buildx and login to Docker Hub, respectively so
that we can build and push the Docker image in the **seventh** step. Finally, the **last** step deploys the stack to the
Swarm cluster.

In order to run the workflow, we need to add the secrets to the repository. Go to the repository settings, under **Security**,
click **Secrets and Variables** and then click **Actions**. Add the following secrets:

![GitHub Secrets](/assets/images/swarm-cicd/github-secrets.png)

Once you have set up the secrets, you can push your changes to the repository and the
workflow should run automatically deploying the app to the Swarm cluster.

![GitHub Actions](/assets/images/swarm-cicd/github-actions.png)

That's it! You have successfully automated the deployment process of your Rails app
to a Docker Swarm cluster on a remote VPS using GitHub Actions. Hopefullly, this post was helpful to you.
If you have any questions or suggestions, feel free to reach out to me on [Twitter](https://twitter.com/rmontas).
