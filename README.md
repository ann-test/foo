# domain service
Welcome to the Domain service! The primary responsibilities of this codebase are to:
1. Cache user membership information (identity, groups, etc.).
2. Provide context information to be used for authorization (in tandem with the Permissions Service).

Until the creation of this service, CircleCI relied exclusively on VCS providers to determine membership. This service enables CircleCI to retrieve membership from other systems (e.g. LDAP).

[![CircleCI](https://circleci.com/gh/circleci/domain-service.svg?style=shield&circle-token=7c37eff8f5e8b7ea0f88911575b8b124e7dc29e4)](https://circleci.com/gh/circleci/domain-service)
[![codecov](https://codecov.io/gh/circleci/domain-service/branch/master/graph/badge.svg?token=MBriDYQQ61)](https://codecov.io/gh/circleci/domain-service)


## Contents
- [Service](#server)
    - [Running Locally](#running-service-locally)
        - [REPL](#repl)
        - [Poking](#poking)
    - [Development](#development)
        - [Dev Namespace (You're going to want to read this)](#dev-namespace)
    - [Data Models](#data-models)
    - [Redis Cache](#redis-cache)
        - [Data](#data)
        - [How to Clear Cache](#how-to-clear-cache)
- [Client](#client)
    - [Running Locally](#running-client-locally)
- [Guidelines](#Guidelines)
    - [Coding Style](#coding-style)
    - [Component Variables](#components-variables)
    - [Passing Components](#passing-components)


## Service
### Running service locally

You can run the domain-service with or without circle/circle running. 

From the top-level domain folder (you should not be in `domain-service/service`), if [circle/circle](https://github.com/circleci/circle#running-circleci-locally) is running, use:
``` Shell
$ docker-compose up -d
```

If circle/circle isn't running, use:
``` Shell
$ docker-compose -f docker-compose-independent.yml up -d
```

We use an internal [clojure-service-image](https://github.com/circleci/clojure-service-image), so you may need to log into our [Docker Hub](https://hub.docker.com/?next=https%3A%2F%2Fhub.docker.com%2F) in order to run things locally. In Slack, @bear should be able to set you up with an account (or to direct you to someone that can). Once you're all set up with an account, from the terminal you can run:

``` Shell
$ docker login -u yourusername ---password-stdin
```

Enter your password when prompted, and then re-run the appropriate `docker-compose` command once you're logged in.


#### REPL

Service exposes REPL from running container, so you can just connect to it at `dev.circlehost:6012`

``` Shell
$ lein repl :connect 6012
```

##### Moar REPL
- [Docker Exec](#docker-exec)
- [Accessing Postgres in Docker](#accessing-postgres-in-docker)

##### Docker Exec 
Pro tip: If you want to run a command inside Docker, just use `docker exec`.

For example, to run the Service tests locally:

``` Shell
$ docker ps
``` 

Grab the container name out of the results (most likely `domain_domain-service_1`)

And execute the command!

``` Shell
$ docker exec -it domain_domain-service_1 lein test2junit
```

Et voila! Local tests run.

Check out the `.circleci/config.yml` file to see where/how tests are run on CI.

##### Accessing Postgres in Docker
Similarly, `docker exec` can be used to drill into the dev database.

``` Shell
$ docker exec -it circle_postgres_1 bash 
```

This should plop you into the root (something like `root@1234blah:`). Set your search path and away you go!

``` Shell
$ psql -U root root
$ set search_path=domain_test;
$ \d
=> You guessed it, list of domain_test relations
```

SQL is muy exciting. Write queries to your heart's content.

### Poking

``` Shell
$ curl dev.circlehost:3012
```
#### Moar Poking
- [HTTPie](#HTTPie)
- [Getting test data to poke with](#Getting-test-data-to-poke-with)

#### HTTPie
You don't need to use it, but omg [HTTPie](https://httpie.org/) makes command line curling more pleasant.

These two puppies will return the same results, but the `http` version will be all beautifully formatted with status information at no extra finger-motion costs to you.

```curl -X get "localhost:3012/contexts?org_id=1527036f-33ee-4a40-ae2f-78ac07450979"```

```http get "localhost:3012/contexts?org_id=1527036f-33ee-4a40-ae2f-78ac07450979"```

#### Getting test data to poke with
If you're working in the client you'll need to generate some of the models manually.

First things first, if you're trying to generate some test data, look in the [Client](https://github.com/circleci/domain-service/tree/master/client/src/circleci/domain/client) to see if you can make a post to build the data you need. (If you see a `new-something` function in the namespace that you want, such as `new-context` in the `contexts` ns, that means you should be good to make a post.)

Keep in mind there are underlying database dependencies, so you'll need an org before you can make a context with an org-id, etc.

There are some pieces of data (e.g. orgs) you'll need to be more creative with because at the time of this writing there are no endpoints that allow you to generate them.

To get a remote user or org-id:
1. Make sure you're running circle/circle in Docker
2. Connect to the circle/circle REPL (at the time of this writing, that means `lein :repl connect 6005`)
2. Log in at https://dev.circlehost:4443/ (this will add you as a user)
3. Grab your user information (`(circle.model.user/all)` ought to do the trick)
4. Get your `analytics-id` key out of the output
5. Use your user-id to make requests
e.g. `http get "localhost:3012/users/c0251332-c41f-4066-ad84-2c4eebe6ceba"`
6. Once you have the http response to your user-id, it should have some projects associated. They should have org-ids. Use those.

**Pro Tip**
Remember to look in the `domain-service` docker logs. They'll be outputting valuable information every time you hit the service.

### Development
- [Dev namespace](#dev-namespace)
- [Access from the REPL](#access-from-the-repl)
- [Inspect current System state](#inspect-current-System-state)
- [Restart the System](#restart-the-System)
- [History](#History)

#### Dev namespace

Service relies heavily on components. The `dev` namespace has several commands that reload whole service in seconds. These will spare you having to kill docker, `docker-compose build` and `docker-compose up` repeatedly.

You could use this namespace to restart the whole system after every change you make, but especially remember to do so (using `(dev/reset)`) if it seems like your change has not registered. 

##### Access from the REPL
You can go there from any other namespace in REPL as:

``` Clojure
any-ns> (user/dev)
#namespace[dev]
dev> 
```

##### Inspect current System state
To inspect current state of the system, access the var `system` in `dev` ns

``` Clojure
any-ns> (keys dev/system)
(:config :web-server :db :rmq :app :providers :vm-manager :health-checker)
any-ns> (clojure.pprint/pprint (:rmq dev/system))
{:conn-spec
 {:host "rabbitmq",
  :port 5672,
  :username "guest",
  :password "guest",
  :vhost "/"},
 :conn
 #object[com.novemberain.langohr.Connection 0x79804a6c "amqp://guest@172.18.0.6:5672/"]}
```
##### Restart the System
To restart the whole system run:

``` Clojure
any-ns> (dev/reset)
:reloading (circleci.vm-service.providers circleci.vm-service.common.config circleci.vm-service.docker-engine circleci.vm-service.common.data-utils circleci.vm-service.db circleci.vm-service.vms-db circleci.vm-service.rabbitmq circleci.vm-service.errors circleci.vm-service.vms circleci.vm-service.common.ring-utils circleci.vm-service.handler circleci.vm-service.main circleci.vm-service.main-test dev user)
#<SystemMap>
```

Note: This does not run database migrations.

##### History
To learn more about components and reloading workflow read/watch:

 1. https://www.youtube.com/watch?v=13cmHf_kt-Q
 2. https://www.youtube.com/watch?v=-RaFcpNiYCo
 3. http://thinkrelevance.com/blog/2013/06/04/clojure-workflow-reloaded

#### Domain model
Domain model from a perspective of domain-service:
![domain](/uml/domain.png)

##### Providers

Providers are an abstraction for any 3rd party service which provides us with identities, orgs, projects, groups or files (configs). Read more justification in the [design doc](https://docs.google.com/document/d/15z4a5FkU38kOpqkKMgZggBy-2gt0METn0W28d2t76U8/edit?usp=sharing).

##### Account-level Providers (auth-layers)

Account-level admins can configure accounts so that all members must be authenticated through a particular Provider (as a so-called auth-layer). In addition to providing authentication, this account-level provider may also be a source of Groups (this is true for LDAP configurations). Users will have a unique Identity associated with this provider.

In that case in order to connect user's identity to that account we need to make sure that one provider might be an auth-layer for only one account. Therefore, we have a uniqueness constraint for `provider_id` in `auth_layers` table.

Although, this creates a certain limitation. For Cloud right now we have three Providers - `github`, `bitbucket` and `opendid-connect`, which are going to be shared among many Orgs and Accounts. Therefore they cannot be used as auth-layers. LDAP/Okta-like providers are going to be unique for each account, thus no problem here. If anytime in the future we would need to use github/bitucket as auth-layer we might need to a) reconsider this uniqueness constraint or b) start creating separate providers for github/bitbucket auth-layers (which would also require creating separate identities for users).

##### Providers API

###### Commit info
Request:
``` Shell
$ http "dev.circlehost:3012/providers/commits/576662fad535f329761e498e82dcc6080e72049d?repo=circleci/circle&provider_id=bcc68be8-ef10-4dd6-9b76-34f19e0db930&credentials=eyJhY2Nlc3MtdG9rZW4iOiJIQUhBIn0"
```

Response body:
``` JSON
{
    "author": {
        "date": "2018-04-10T19:44:59Z",
        "email": "conor@mcdermottroe.com",
        "name": "Conor McDermottroe"
    },
    "body": "Keep the SOCKS proxy alive for as long as possible",
    "committer": {
        "date": "2018-04-10T19:44:59Z",
        "email": "noreply@github.com",
        "name": "GitHub"
    },
    "id": "906ed0d0d1a399b2a3182b07b5f687bbe4e5744e",
    "subject": "Merge pull request #8734 from circleci/keep-socks-proxy-alive",
    "url": "https://github.com/circleci/circle/commit/906ed0d0d1a399b2a3182b07b5f687bbe4e5744e"
}
```

Parameters:
 * `repo` - repository name including username/org
 * `provider_id` - provider id
 * `credentials` - base64 encoded map with credentials as they are stored in identities collection/table

#### Data models

The service is manipulated with a couple of data models. Several are documented here, though some may be missing. It's worth checking the values in the [database](#accessing-postgres-in-docker) if you're concerned about missing a few.

- [Identity data](#Identity-data)
- [Contexts](#contexts)
- [Circle data](#Circle-data)
- [Domain data](#Domain-data)
- [Group](#Group)
- [Org](#Org)
- [Plans](#Plans)
- [Project](#Project)

##### Identity data

Unified view of data received from identity providers (orgs/projects/groups).
Every implementation of `IdentityProvider` protocol should return data in this format.

All IDs in this model are `external` because not owned by us and provided by various identity providers.

###### Contexts
``` Clojure
{:id "da5c615d-748d-4166-ac71-bcd17d5e26e9""
 :name "context-name"
 :org-id "80b006bf-29d8-44bb-bc33-fb01d96ca332""
 :level "org"
 :account-id "12bd5f26-8a8e-4fd8-938e-0a1613a61e1d""}
```

###### Circle data

Data received from circle/circle.

###### Domain data

Final representation of users/orgs/projects/groups that service store in DB/cache and return to requests.

###### Group

``` Clojure
{:external-id "123456"
 :name "group-name"
 :org-id "12bd5f26-8a8e-4fd8-938e-0a1613a61e1d"
 :account-id nil
 :provider-id "fdb2f2b4-2195-4c8f-a965-07a111dcbe49"
 :name "development}
```

###### Org

``` Clojure
{:id "1145736f-f992-4b1a-9d32-8d2b4c0639f9"
 :external-id "123456"
 :account-id "80b006bf-29d8-44bb-bc33-fb01d96ca332"
 :name "org-name"
 :permissions {:admin true}
 :provider-id "fdb2f2b4-2195-4c8f-a965-07a111dcbe49}
```

###### Plans

``` Clojure
{:id "123456"
 :org-id "78910"}
```

###### Project

``` Clojure
{:id "45c070fc-86c1-4142-9d9c-957eb5c3b4f6"
 :external-id "12345"
 :org-id "d8cac30f-a741-45ab-9f49-a77c1146968e"
 :name "project-name"
 :permissions {:admin true :read true :write true}
 :owner {:external-id "987665"
         :name "owner-name"}
 :provider-id "fdb2f2b4-2195-4c8f-a965-07a111dcbe49"}
```

#### Redis cache
All information in Redis is stored in key-value pairs.
- Users are cached with their ID as the key-value.
- The active users for a given org are cached with their IDs in a set. The org-id is the key for each set of logged in users.

##### Data
It might be helpful to visualize the data like this:
``` Shell
$ user-id-1   user-1-data
$ user-id-2   user-2-data
$ org-id-1    [user-id-1 user-id-2 user-id-3]
$ user-id-3   user-3-data
$ org-id-2    [user-id-1]
```

##### How to Clear Cache
To clear the caches for all of the users of a given org, use the [`delete-users-cache-for-org`](https://github.com/circleci/domain-service/blob/master/client/src/circleci/domain/client/orgs.clj#L8) endpoint. Your request will look something like:

```http delete "localhost:3012/orgs/1527036f-33ee-4a40-ae2f-78ac07450979/users"```


To clear the cache for a specific user, use the [`delete-user-cache`](https://github.com/circleci/domain-service/blob/master/client/src/circleci/domain/client/users.clj#L18) endpoint. Your request will look something like:

```http delete "localhost:3012/users/1527036f-33ee-4a40-ae2f-78ac07450979"```

If you need to clear everything, cycle the Redis pod. The Redis uri can be found in our Kubernetes config file. You'll need to have [K8 up and running](https://github.com/circleci/engineering/blob/master/howto/ssh.md#to-kubernetes), then you can find the config on the [domain-service k8 dashboard](http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/#!/secret/default/domain-service-config-v1.0?namespace=default). 

The Domain Redis dashboard can be found be found [here](http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/#!/deployment/default/domain-redis?namespace=default).

## Client

### Running Client locally

To start the Client REPL, cd into the Client folder and run: `lein repl :headless`.

This will start a repl, and the output will include the port number that you can then connect to on the localhost.


## Guidelines

#### Coding style
This service extends the common Clojure style guide with a few additional items related to components. 

Because components are heavily used and passed around, it's useful to have some guidelines for them.

##### Components variables 
Variable name should have format `<component-name>` (name wrapped in opening and closing arrowheads). For example:
    
``` Clojure
(defn update-vm-status
  [<db> id status]
  ;; ...
)
```

##### Passing components to a function
Component should always be the first argument in a function arg list.
