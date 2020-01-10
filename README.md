# Mission Control API

A GraphQL API with data model for managing products, projects, people and roles

## The Production Stack

The production deployment of this API is built as a stack of technologies, from top to bottom...

* AWS
  * Handles networking (ALB, VPC, etc.) and container management (ECS)
* Apollo Server 2
  * Provides a GraphQL server for resolvers, which is where business logic lives
* Prisma
  * Provides an ORM to translate from Graphql to Postgres, Apollo resolvers mainly call a Prisma Client to access data
* Postgres
  * Provides persistent storage for data, this is managed by AWS RDS in production but is run in a container during local development

## Directories

The various layers are organized in their own folders to help keep everything neat and tidy.

### Root

Most of the time you'll be working in the `prisma` or `apollo` directories, messing with those layers. The root has a couple globally relevant files

* `docker-compose.yml` This file declares the system and how the various components interact
* `sourceme.sh` a script that loads up environment variables for ez development (Note: As implied, you need to source this file `source ./sourceme.sh`)

### Cloudformation

This directory contains a collection of AWS Cloudformation templates for deploying and configuring the various AWS resource that make the system work.

Usage: The system is broken into several different stacks to help speed up deployment and updating:

* `network.cf.yaml` This stack contains the main network components for the VPC.
* `lb.cf.yaml` This stack contains the various load balancers.
* `dns.cf.yaml` This stack contains the DNS for the system
* `rds.cf.yaml` This stack contains the RDS database for the system

The stack parameters are contained in a JSON file names `<stack>-params.json` and are passed to the AWS CLI using the shell script. This is a about as simple as Cloudformation gets without using a 3rd party library. Oh well... 🙁

### Apollo Server

The Apollo Server is a Dockerized version of Apollo where all of the various business logic and schemas will live. This layer provides the outward facing API for clients to use.

* `src` Source code for resolvers and various other components
* `src\generated\prisma-client` The generated Prisma client
* `schema` Schema files for Apollo to use for its API
* `schema\generated\prisma.graphql` The schema generated by Prisma, which can be used by Apollo to ensure the Apollo model matches the Prisma model

### Prisma

The Prisma server translates GraphQL queries into Postgres queries and managed everything about the database

* `prisma.yml` The main Prisma configuration file.
* `prisma-datamodel.graphql` The Prisma data model for building the database
* `prisma-seed-*.graphql` Seed files for filling the database with data
  
## Development Flow

### Step 1) Export environment variables

There are a bunch of variables that are _not_ hardcoded into the source code. These are pulled from the environment when this is run locally or in production. The easiest way to do this is to create a `.env` file in the root directory of the project. Being absolutely sure the `.env` is listed in your `.gitignore` file so that these variables _never_ get committed into the repository.

Assuming you're using Okta as your Oauth provider, the `.env` file should look something like this:

```text
OAUTH_TOKEN_ENDPOINT=<https://_your okta domain_/oauth2/default>
OAUTH_CLIENT_ID=<_your okta client id_>

PRISMA_MANAGEMENT_API_SECRET=<_a random secret to secure the prisma management api_>

PRISMA_ENDPOINT=<_when running locally, something like: http://localhost:7000/mc/local_>
PRISMA_SECRET=<_a random secret to secure the prisma service api_>
```

Once you get your `.env` file setup, you can run `source ./sourceme.sh` to export them. This file is also picked up by Docker Compose automatically.

### Step 2) Launch locally using Docker Compose

`docker-compose up --build`

Note: You'll need to get Docker installed on your machine (<https://www.docker.com/>)

If everything works, you'll see that all of the services are now up and running localling on your machine:

* Apollo is listening on <http://localhost:8000>
* Prisma is listening on <http://localhost:7000>
* Postgres is listening on <localhost:5432>

### Step 3) Deploy the Prisma Service

```sh
cd prisma
prisma deploy
prisma seed
```

This step will tell Prisma to deploy the datamodel `prisma-datamodel.graphql' to Postgres and seed in the initial data. At this point, Prisma will create all of the tables and relations required to store the data that's been defined in the data model. You can check this out by going to either <http://localhost:7000/> or <http://localhost:7000/_admin> in your browser.

If you do talk to Prisma using your browser, you'll need to pass it a token. You create a token using the `prisma token` command, which uses the `PRISMA_SECRET` environment variable to create a token that you can copy into either GraphQL Playground or the Prisma Admin.

More details <prisma/README.md>

Notice that the Prisma API allows you to do just about anything you want to the data if you have a token. You can create, read, update and delete everything. Convenient, but not very secure... which is why we need another layer to handle all the detailed business logic to protect our data. Apollo!

### Step 4) Build Apollo

Prisma is the data layer, it doesn't have any business logic for restricting what happens to the data, it just facilitates data passing in and our of our database. Apollo is where the 'backend' logic lives, mainly in the form of resolver functions.

See <apollo/README.md>