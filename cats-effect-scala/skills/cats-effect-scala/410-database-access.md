# Database Access

[doobie](https://typelevel.org/doobie/) is a functional JDBC layer for Cats
Effect. It does not hide SQL — you write SQL, doobie handles the plumbing,
resource safety, and type mapping.

## Dependencies

```sbt
val doobieVersion = "1.0.0-RC13"

libraryDependencies ++= Seq(
  "org.typelevel" %% "doobie-core"     % doobieVersion,
  "org.typelevel" %% "doobie-hikari"   % doobieVersion,
  "org.typelevel" %% "doobie-postgres" % doobieVersion,
  "org.typelevel" %% "doobie-munit"    % doobieVersion % Test,
)
```

> **Important:** doobie moved from groupId `org.tpolecat` to `org.typelevel` in
> `1.0.0-RC13`, and its packages moved from `doobie.*` to
> `org.typelevel.doobie.*`. Tutorials and Stack Overflow answers written before
> that release show the old coordinates and imports — both must change
> together.

## The Transactor

A `Transactor[F]` knows how to get a `Connection` and run a `ConnectionIO`. In
production it wraps a HikariCP pool and is a `Resource`:

```scala
import cats.effect.{IO, Resource}
import org.typelevel.doobie.*
import org.typelevel.doobie.hikari.HikariTransactor

def transactor(config: DbConfig): Resource[IO, HikariTransactor[IO]] =
  for
    ec <- ExecutionContexts.fixedThreadPool[IO](config.poolSize)
    xa <- HikariTransactor.newHikariTransactor[IO](
            driverClassName = "org.postgresql.Driver",
            url             = config.url,
            user            = config.user,
            pass            = config.password.value,
            connectEC       = ec,
          )
  yield xa
```

> **Warning:** `Transactor.fromDriverManager` opens a new connection per query
> and has no pool. It is fine for scripts and tests, never for a service.

## Queries

```scala
import org.typelevel.doobie.*
import org.typelevel.doobie.implicits.*

final case class User(id: UUID, name: String, email: String)

// One row expected:
def find(id: UUID): ConnectionIO[Option[User]] =
  sql"SELECT id, name, email FROM users WHERE id = $id"
    .query[User]
    .option

// Many rows:
def byStatus(status: String): ConnectionIO[List[User]] =
  sql"SELECT id, name, email FROM users WHERE status = $status"
    .query[User]
    .to[List]

// Exactly one, error otherwise:
def count: ConnectionIO[Long] =
  sql"SELECT count(*) FROM users".query[Long].unique
```

Interpolated values become **bind parameters**, not string concatenation — the
`sql` interpolator is SQL-injection-safe by construction.

> **Warning:** the one thing the interpolator cannot parameterise is an
> identifier (a table or column name). Never interpolate a user-supplied
> column name via `Fragment.const` — validate it against an allow-list first.

Result rows map to case classes positionally, so column order in the `SELECT`
must match the field order in the class.

## Updates

```scala
def insert(user: User): ConnectionIO[Int] =
  sql"""INSERT INTO users (id, name, email)
        VALUES (${user.id}, ${user.name}, ${user.email})"""
    .update
    .run

// Get the generated key back:
def insertReturning(name: String, email: String): ConnectionIO[User] =
  sql"INSERT INTO users (name, email) VALUES ($name, $email)"
    .update
    .withUniqueGeneratedKeys[User]("id", "name", "email")

// Batch — one prepared statement, many parameter sets:
def insertMany(users: List[User]): ConnectionIO[Int] =
  Update[User]("INSERT INTO users (id, name, email) VALUES (?, ?, ?)")
    .updateMany(users)
```

## Transactions

Everything composed in `ConnectionIO` runs in **one transaction**, committed
when `.transact` completes and rolled back on error or cancellation:

```scala
def transferFunds(from: AccountId, to: AccountId, amount: Money): IO[Unit] =
  val work: ConnectionIO[Unit] =
    for
      _ <- debit(from, amount)
      _ <- credit(to, amount)
      _ <- recordAudit(from, to, amount)
    yield ()
  work.transact(xa)
```

> **Important:** the transaction boundary is `.transact`, not the individual
> queries. Calling `.transact` inside a loop gives you N transactions. Build
> the whole unit of work in `ConnectionIO` first, then transact once.

Repositories should therefore return `ConnectionIO`, letting the *service*
decide the transaction boundary:

```scala
trait UserRepo:
  def find(id: UUID): ConnectionIO[Option[User]]
  def insert(user: User): ConnectionIO[Int]

class UserService(repo: UserRepo, xa: Transactor[IO]):
  def register(cmd: CreateUser): IO[User] =
    val work: ConnectionIO[User] =
      for
        existing <- repo.findByEmail(cmd.email)
        user     <- existing.fold(repo.insertReturning(cmd))(_ =>
                      MonadThrow[ConnectionIO].raiseError(EmailTaken(cmd.email)))
      yield user
    work.transact(xa)   // one transaction; rolled back if EmailTaken is raised
```

## Dynamic queries

`Fragment`s compose safely:

```scala
import cats.syntax.all.*
import org.typelevel.doobie.Fragments

def search(name: Option[String], status: Option[String]): ConnectionIO[List[User]] =
  val base = fr"SELECT id, name, email FROM users"
  val filters = List(
    name.map(n => fr"name ILIKE ${s"%$n%"}"),
    status.map(s => fr"status = $s"),
  ).flatten
  (base ++ Fragments.whereAndOpt(filters*)).query[User].to[List]
```

`Fragments` also provides `in`, `andOpt`, `orOpt`, `setOpt` for the common
shapes.

## Streaming large result sets

```scala
import fs2.Stream

def streamAll: Stream[IO, User] =
  sql"SELECT id, name, email FROM users"
    .query[User]
    .stream            // Stream[ConnectionIO, User]
    .transact(xa)      // Stream[IO, User]
```

The cursor stays open and rows arrive in chunks, so memory is constant. Pair it
with an http4s streaming response for exports of any size.

## Type mappings

doobie knows JDBC's standard types. For your own:

```scala
opaque type UserId = UUID

object UserId:
  given Meta[UserId] = Meta[UUID].imap(UserId.apply)(_.value)

// Enums as strings:
given Meta[Status] = Meta[String].timap(Status.valueOf)(_.toString)

// PostgreSQL-specific types come from doobie-postgres:
import org.typelevel.doobie.postgres.implicits.*   // arrays, json, inet, ...
```

## Checking queries at test time

doobie can verify every query against the live schema — column types,
nullability, and arity:

```scala
import org.typelevel.doobie.munit.IOChecker

class QuerySuite extends munit.FunSuite with IOChecker:
  override val transactor: Transactor[IO] = testTransactor

  test("find")   { check(UserRepo.findQuery(UUID.randomUUID())) }
  test("insert") { check(UserRepo.insertUpdate(sampleUser)) }
```

This catches a renamed column at test time rather than in production. Run it
against a Testcontainers Postgres instance.

## Blocking and the connection pool

doobie runs JDBC calls on the blocking pool for you — you do not need to wrap
`.transact` in `IO.blocking`. But the connection pool is a hard concurrency
limit: 100 concurrent `parTraverse`d queries against a 10-connection pool will
queue. Size `parTraverseN` to the pool, not the other way round.

## skunk — when you want Postgres without JDBC

[skunk](https://typelevel.org/skunk/) speaks the Postgres wire protocol
directly, so it is fully non-blocking and has no JDBC or connection-pool
machinery:

```sbt
libraryDependencies += "org.tpolecat" %% "skunk-core" % "0.6.5"
```

```scala
import skunk.*
import skunk.codec.all.*
import skunk.implicits.*

val session: Resource[IO, Session[IO]] =
  Session.single[IO](host = "localhost", port = 5432,
                     user = "postgres", database = "app", password = Some(pw))

val selectUser: Query[UUID, User] =
  sql"SELECT id, name, email FROM users WHERE id = $uuid"
    .query(uuid *: varchar *: varchar)
    .to[User]

session.use(_.prepare(selectUser).flatMap(_.option(id)))
```

Choose skunk for Postgres-only projects that want end-to-end non-blocking I/O
and precise, explicit codecs. Choose doobie when you need JDBC (other
databases, existing drivers, or a shared pool) or prefer inferred row mapping.
