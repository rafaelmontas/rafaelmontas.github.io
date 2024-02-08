---
layout: post
title: "Deploy a Rails app to a Docker Swarm cluster on Hetzner"
header: "Deploy a Rails app to a Docker Swarm cluster on Hetzner"
slug: "deploy-rails-docker-swarm-hetzner"
excerpt: "Recently, I've been working on a SaaS platform that I'm building using Ruby on Rails. I decided to use Hetzner for hosting and Docker Swarm to handle the containers orchestration in production."
image: /assets/images/method-lookup-path.png
---

In this post, I'll show you how to set up a Docker Swarm cluster on Hetzner and deploy
a Rails app to it. We will be seting up two hosts, one for the web server acting
as manager and the other for the database acting as worker.

First, let's create a new Rails app and prepare the database:

```bash
$ rails new docker-swarm -a propshaft -d postgresql -c tailwind
$ rails db:prepare
```

You should now be able to run the app locally with `rails s` and see the welcome page.

![Rails welcome page](/assets/images/docker-swarm-hetzner/rails-welcome.png)

Just so we can test our database connection and data persistency in the swarm, let's scaffold
a simple Post resource and set the index action as the root route. :

```bash
$ rails g scaffold Post title:string
$ rails db:migrate
```

```ruby
root "posts#index"
```

## Provisioning the servers on Hetzner

We can do this through the Hetzner Cloud Console but we will use the hcloud CLI.
First, you need to [create an account](https://accounts.hetzner.com/login) on Hetzner, create
a project, and [generate an API token](https://docs.hetzner.com/cloud/api/getting-started/generating-api-token/)
(with read and write permissions). Then, install the [hcloud CLI](https://community.hetzner.com/tutorials/howto-hcloud-cli)
with `brew install hcloud` on macOS.

Once you have the token, you can authenticate with the CLI by creating a context:

```bash
$ hcloud context create swarm-hetzner
```

You will be prompted to enter the API token and the context will be created. Now, you can start
provisioning the two servers.

In order for us to be able to SSH into the servers, we need to create an SSH key in our local machine
and then assocaite it with Hetzner so we can use it when creating the servers:

```bash
$ ssh-keygen -f ~/.ssh/id_rsa_personal
$ hcloud ssh-key create --name personal-key --public-key-from-file ~/.ssh/id_rsa_personal.pub
```

You can list the SSH keys to make sure it was added successfully:

```bash
$ hcloud ssh-key list

ID         NAME           FINGERPRINT                                       AGE
19362876   personal-key   cf:0b:89:00:00:5f:7d:e5:93:c8:17:16:68:64:38:b3   55s
```

Now, we can create the two servers. We will use the `cx11` type for both, which is the cheapest
one. You can choose a different type if you want:

```bash
$ hcloud server create --name app-manager --type cx11 --image docker-ce --ssh-key 19362876
$ hcloud server create --name db-worker --type cx11 --image docker-ce --ssh-key 19362876
```

You should see the servers listed when you run `hcloud server list`:

```bash
$ hcloud server list

ID         NAME          STATUS    IPV4             IPV6                      PRIVATE NET   DATACENTER   AGE
43107995   app-manager   running   37.27.34.249     2a01:4f9:c012:8df4::/64   -             hel1-dc2     4m
43108038   db-worker     running   95.217.128.149   2a01:4f9:c011:94ac::/64   -             hel1-dc2     1m
```

Optionally, you can go on and add a new entry to the config file on your local device. With this you will be
able to simply use `ssh <unique-name>` instead of `ssh <username>@<IP-address>` to connect to the server:

```bash
$ nano ~/.ssh/config

Host swarm-manager
  HostName 37.27.34.249
  User root
  IdentityFile ~/.ssh/id_rsa_personal
  PreferredAuthentications publickey
```
Now you can connect to the server with `ssh swarm-manager` and verify that Docker is installed:

```bash
$ ssh swarm-manager
$ docker --version

>> Docker version 25.0.1, build 29cf629
```

## Adding firewalls to both servers

