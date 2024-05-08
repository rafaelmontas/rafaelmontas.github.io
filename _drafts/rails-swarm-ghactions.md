---
layout: post
title: "Automate app deployment with GitHub Actions and Docker Swarm - CI/CD"
header: "Automate app deployment with GitHub Actions and Docker Swarm - CI/CD"
slug: "automate-app-deployment-with-github-actions"
excerpt: "Once you can deploy an app to a remote Docker Swarm cluster from your local machine, you might want automate the process so that your app gets deployed every time you push your code to your repository."
image: /assets/images/docker-swarm-hetzner/swarm-hetzner.jpg
---

This is the second part of a series of posts on deploying a Rails app to a Docker Swarm
cluster on Hetzner Cloud. In the first part, we set up the Swarm cluster on
Hetzner Cloud and deployed a Rails app to it from our local machine. If you haven't read
it yet, you can find it [here](/deploy-rails-docker-swarm-hetzner). In this post, we'll
automate the  deployment process with GitHub Actions, so that every time you push your
code to GitHub a CI/CD pipeline will lint, test, build and deploy your app to the Swarm.

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
repository but definitely need them to deploy the app. In order to handle this, we will use
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

Wi will implement a rake task to fetch the secrets from Bitwarden and inject them into the `.env` file.
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
concept of a **Workflow** which is defined by a YAML file in the `.github/workflows` direcory.

We are going to configure a GitHub Actions workflow to be triggered when we push our changes
to the repository, running our Test suite and deploying the app to the VPS using
Docker Swarm.

First, let's create the `.github/workflows` directory and create a new file (workflow).
Let's call it `deploy.yml`.
