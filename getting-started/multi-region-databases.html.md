---
title: "Multi-region PostgreSQL"
layout: docs
sitemap: false
nav: firecracker
---

Most read heavy, PostgreSQL backed applications work natively across regions on Fly.io, no architectural changes required. Deploying an app and database in multiple regions takes advantage of two Fly features:

1. Regional read replicas
1. The `fly-replay` header

With regional read replicas configured, the `fly-replay` headers lets you specify exactly which requests need to be serviced by the primary, writable database. When we detect this header, we will replay the entire request to the region you specify. It looks something like this:

<figure class="w:full mb:4">
  <img class="m:0" src="/public/images/fly-global-postgres.svg"  alt="Diagram of app + global postgres on Fly.io">
</figure>

In most runtimes, it's straightforward to catch a read-only database error in a replica region and serve a response with the appropriate replay header.

This guide is all about PostgreSQL, but the deployment topology will work with MySQL, MongoDB, and any other database with read replica support.

## Create a PostgreSQL cluster

If you don't already have a PostgreSQL cluster running, you can create one with the `fly` CLI:

```
fly pg create --name chaos-postgres --region scl
```

This creates a two node PostgreSQL cluster in Santiago Chile, one leader for writes, one replica for redundancy.

## Add read replicas

Adding read replicas is simple, just create more disks:

```
fly volumes create pg_data -a chaos-postgres --size 10 --region atl
fly volumes create pg_data -a chaos-postgres --size 10 --region ord
fly volumes create pg_data -a chaos-postgres --size 10 --region ams
fly volumes create pg_data -a chaos-postgres --size 10 --region syd
```

Then, add more VMs:

```
fly scale count 6 -a chaos-postgres
```

The `chaos-postgres` cluster will now have read replicas in Atlanta, Chicago, Amsterdam, and Sydney. When you run `fly status -a chaos-postgres` you should see output like this:

```text
ID       VERSION REGION DESIRED STATUS
240eb1cd 2       ams    run     running (replica)
83b849fa 2       ord    run     running (replica)
e759c2ed 2       atl    run     running (replica)
d8e8a317 2       syd    run     running (replica)
d8e8a317 2       scl    run     running (replica)
987f4b41 2       scl    run     running (leader)
```

## Configure connection strings

### Attach database to application
To hook your app up to your cluster, run the `attach` command from your application directory:

```
fly pg attach --postgres-app chaos-postgres
```

This installs a `DATABASE_URL` secret in your application, which is available to your app processes as an environment variable. The command also prints the connection string to the console.

### Connect to regional replicas

The generated connection string uses port `5432` to connect to PostgreSQL. This port always forwards you to a writable instance. Port `5433` is direct to the PostgreSQL member, and used to connect to read replicas directly.

You can use the proxy port (`5432`) to connect from every region, but it will be quite slow. Especially because we put our PostgreSQL cluster in Santiago. Connecting to "local" replicas is much quicker, but does take some app logic.

The basic logic to connect is:

1. Set a `PRIMARY_REGION` environment variable on your app, `scl` for our `chaos-postgres` cluster.
2. Check the `FLY_REGION` environment variable at connect time, use `DATABASE_URL` as is when `FLY_REGION=scl`
3. Modify the `DATABASE_URL` when running in other regions:
   1. Change the hostname to `<region>.chaos-postgres.internal`
   2. Change the port to `5433`

This is what it looks like in Ruby:

```ruby
class Fly
  def self.database_url
    primary = ENV["PRIMARY_REGION"]
    current = ENV["FLY_REGION"]
    db_url = ENV["DATABASE_URL"]
    
    if primary.blank? || current.blank? || primary == current
      return db_url
    end
    
    u = URI.parse(db_url)
    u.hostname = "#{current}.#{u.hostname}"
    u.port = 5433
    
    return url.to_s
  end
end
```

Running this in `scl` will use the built-in `DATABASE_URL` and connect to port `5432`:

```
postgres://<user>:<password>@chaos-postgres.internal:5432/rails_on_fly?sslmode=disable
```

In the other regions, the app will use regional URLs and connect to port `5433`:
```
postgres://<user>:<password>@ord.chaos-postgres.internal:5433/rails_on_fly?sslmode=disable
postgres://<user>:<password>@atl.chaos-postgres.internal:5433/rails_on_fly?sslmode=disable
postgres://<user>:<password>@ams.chaos-postgres.internal:5433/rails_on_fly?sslmode=disable
postgres://<user>:<password>@syd.chaos-postgres.internal:5433/rails_on_fly?sslmode=disable
```

## Detect write requests

### Catch read-only errors
PostgreSQL conveniently sends a "read only transaction" error when you attempt to write to a read replica. All you need to do to detect write requests is catch this error.

### Replay the request

Once caught, just send a `fly-replay` header specifying the primary region. For `chaos-postgres`, send `fly-replay: region=scl`, and we'll take care of the rest.

If you're working in Rails, just add this to your `ApplicationController`:

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::StatementInvalid do |e|
    if e.cause.is_a?(PG::ReadOnlySqlTransaction)
      r = ENV["PRIMARY_REGION"]
      response.headers["fly-replay"] = "region=#{r}"
      Rails.logger.info "Replaying request in #{r}"
      render plain: "retry in region #{r}", status: 409
    else
      raise e
    end
  end
end
```

## Library support

We would like to build libraries to make this seamless for most application frameworks and runtimes. If you have a particular app you'd like to distribute with PostgreSQL, [post in our community forums](https://community.fly.io/t/multi-region-database-guide/1600) and we'll write some code for you.

## Consistency model

This is a fairly typical read replica model. Read replicas are usually eventually consistent, and can fall behind the leader. Running read replicas across the world _can_ exacerbate this effect and make read replicas stale more frequently.

### Request with writes

Requests to the primary region are strongly consistent. When you use the replay header to target a particular region, the entire request runs against the leader database. Your application will behave like you expect.

### Read only requests

Most apps accept a `POST` or `PUT`, do a bunch of writes, and then redirect the user to a `GET` request. In most cases, the database will replicate the changes before the user makes the second request. But not always!

Most read heavy applications aren't especially sensitive to stale data on subsequent requests. A lagging read replica might result in an out of date view for users, but this might be reasonable for your use case.

If your app is sensitive to this (meaning, you never, under any circumstances want to show users stale data), you should be careful using read replicas.

### Managing eventual consistency

For apps that are sensitive to consistency issues, you can add a counter or timestamp to user sessions that indicates what "version" of the database a particular user is expecting. When the user makes a request and the session's data version differs from the replica, you can use the same `fly-replay` header to redirect their request to the primary region – and then you'll know it's not stale.

In theory, you _could_ run PostgreSQL with synchronous replication and block until replicas receive writes. This probably won't work well for far flung read replicas.

## This is wrong for some apps

We built this set of features for read heavy apps that are primary HTTP request based. That is, most requests only perform reads and only some requests include writes.

#### Write heavy workloads

If you write to the database on every request, this will not work for you. You will need to make some architectural changes to run a write heavy app in multiple regions.

Some apps write background info like metrics or audit logs on every request, but are otherwise read heavy. If you're running an application like this, you should consider using something like [nats.io](https://nats.io) to send information to your primary region asynchronously.

Truly write heavy apps require latency aware data partitioning, either at the app level or in a database engine. There are lots of interesting new databases that have features for this, try them out!

#### Long lived connections

If your app makes heavy use of long lived connections with interpolated writes, like websockets, this will not work for you. This technique is specific to HTTP request/response based apps that bundle writes up into specific requests.