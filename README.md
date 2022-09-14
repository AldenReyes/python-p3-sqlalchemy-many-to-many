# Many-to-Many Relationships

## Learning Goals

- Use SQLAlchemy to join tables with one-to-one, one-to-many, and
  many-to-many relationships.

***

## Introduction

In the previous lesson, we saw how to create a **one-to-many** association
between two models using SQLAlchemy by following certain naming conventions,
use of the `relationship()` and `backref()` methods, and using the right
foreign key on our tables when generating the migrations.

In the SQL section, we learned about one other kind of relationship: the
**many-to-many**, also known as the **has many through**, relationship. For
instance, in a domain where a **cat** has many **owners** and an **owner** can
also have many **cats**, we needed to create another table to join between those
two tables:

![Pets Database ERD](https://curriculum-content.s3.amazonaws.com/phase-3/sql-table-relations-creating-join-tables/cats-cat_owners-owners.png)

In this lesson, we'll learn how to create a **many-to-many** relationship in
SQLAlchemy. We'll continue working on our games and reviews domain, but this
time we'll add a third model into the mix: a users model. We'll be setting up
these relationships:

- A game **has many** reviews
- A game **has many** users, **through** reviews
- A review **belongs to** a game
- A review **belongs to** a user
- A user **has many** reviews
- A user **has many** games, **through** reviews

Once we're done setting up the database tables, here's what the ERD will look like:

![Game Reviews ERD](https://curriculum-content.s3.amazonaws.com/phase-3/active-record-associations-many-to-many/games-reviews-users-erd.png)

To get started, run `pipenv install` and `pipenv shell`, then follow along with
the code.

***

## Creating a User Model

Right now, we've got code for the `Game` model (and the `games` table), along
with the code for the `Review` model (and the `reviews` table) from the previous
lesson.

To start, let's add the code we'll need for the `User` model as well. Let's
create the `users` table with a `name` column and timestamps:

```py
# app/db.py

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer(), primary_key=True)
    name = Column(String())
    created_at = Column(DateTime(), server_default=func.now())
    updated_at = Column(DateTime(), onupdate=func.now())

    # don't forget your __repr__!
    def __repr__(self):
        return f'User(id={self.id}, ' + \
            f'name={self.name})'
```

Note that for our models' timestamps, we are using some new arguments and
values:

- `server_default` tells the database schema to set a value from the database
  itself. Since the database is kept in one central location, assigning it the
  work of creating default values means that we don't have to worry about the
  quality of our developers' or users' computers.
- `onupdate` means exactly what it says: when the record is updated, the
  column value is set.
- We saw `func` briefly in the previous module; it allows us to use SQL
  operations instead of their Python counterparts. This benefits us for the same
  reasons as `server_default`. `func.now()` is equivalent to the current time.

We'll also need to modify the `reviews` table and add a foreign key to refer to
our `users` table. Remember, each review now **belongs to** a specific user. Any
time we create a **belongs to** relationship, we need a foreign key to establish
this relationship:

```py
# app/db.py

# User model
    reviews = relationship('Review', backref=backref('user'))
```

Let's also edit the `Review` model to add our new foreign key:

```py
# app/db.py

# Review model
    user_id = Column(Integer(), ForeignKey('users.id'))
```

Now run `alembic revision --autogenerate -m'Add User model'` from the
`many-to-many` directory to make our migration. If all goes well, run
`alembic upgrade head` to push your migrations to the database.

```console
$ alembic upgrade head
# => INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
# => INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
# => INFO  [alembic.runtime.migration] Running upgrade  -> 9e396fc70825, Add User model
```

Run the seed file as well to populate the `games` and `reviews` tables:

```console
$ python app/seed.py
```

***

## Creating a Many-to-Many Relationship

There are several ways to approach a many-to-many relationship in SQLAlchemy.
All require some sort of intermediary between the two models. We could do this
using the `Review` model, but this isn't the best choice. What if we wanted to
add functionality in the future where users could log their games without adding
reviews? In any case, it's unlikely that reviews will be the only things tying
users and games to one another.

Many-to-many relationships in SQLAlchemy use intermediaries called **association
tables**. These are tables that exist only to join two related tables together.
This might sound like a waste at first, but keeping the foreign keys,
relationships, and `backref`s confined to this one table allows us to freely
make changes to our related tables later on without worrying too much about how
they might affect each other.

There are two approaches to building these associations: association objects,
which are most similar to the models we've built so far, and the more common
approach, `Table` objects.

### Many-to-Many with an Association Object

An association object is really just another model, so let's add one into
`app/db.py`. The convention is to call this model a combination of the related
models' names, so let's call ours **`GameUser`**.

```py
# app/db.py

class GameUser(Base):
    __tablename__ = "game_users"

    game_id = Column(ForeignKey('games.game_id'), primary_key=True)
    user_id = Column(ForeignKey('users.user_id'), primary_key=True)

    game = relationship('Game', backref=backref('users'))
    user = relationship('User', backref=backref('games'))

    def __repr__(self):
        return f'GameUser(game_id={self.game_id}, ' + \
            f'user_id={self.user_id})'
```

An association object should use a combination of the related models' primary
keys as its own primary key. This allows us to keep the field unique while
also making it clear which two records its records point to.

Next, we use the `relationship()` method to connect to both the `Game` and
`User` models. Note that we refer to the other related table in our `backref()`
instead of `game_users`. We typically choose to hide association tables while
our our application runs; this reference details the two endpoints of the
association rather than the middle.

Finally- don't forget your `__repr__`!

### Many-to-Many with `Table` Objects

`Table` objects are instances of the `sqlalchemy.Table` class. They function
more or less the same as data models, with the exception of being a little
more compact. This syntax de-emphasizes association tables in your models and
is the preferred approach in SQLAlchemy.

Let's build the same association table as above with our new syntax:

```py
# app/db.py

game_user = Table(
    'game_users',
    Base.metadata,
    Column('game_id', ForeignKey('games.game_id'), primary_key=True),
    Column('user_id', ForeignKey('users.user_id'), primary_key=True)
)

# Game
    users = relationship('User', secondary=game_user, back_populates='games')

# User
    games = relationship('Game', secondary=game_user, back_populates='users')
```

Because we are creating an object that is being used in subsequent code,
association tables made from the `Table` class must be defined above other data
models. Otherwise, they wouldn't exist when they were referenced!

A few more notes on this approach:

- The `Game` and `User` models each require a relationship with the other.
- The `secondary` argument refers to the intermediary table in a many-to-many
  relationship.
- The `back_populates` operates similiarly to `backref`, with the exception that
  it must be used on both sides of a relationship. Because many-to-many
  relationships are symmetrical, use of `back_populates` in both models is the
  best way to leave readable code behind for other developers.

Run `alembic revision --autogenerate -m'Add game_user Association Table'`, then
`alembic upgrade head`. You can use the script in `app/seed.py` to generate new
data and interact with your database through the Python shell. To create
relationships between `Game` records and `User` records, open the script and
un-comment the block starting at line 60:

```py
# app/seed.py

if game not in user.games:
    user.games.append(game)
    session.add(user)
    session.commit()
```

This will add a `Game` record to a `User` record's `games` if the user has
logged a review for the game. When the change is committed, SQLAlchemy also
builds the relationship in reverse, adding the `User` record to the `Game`
record's `users`!

***

## Conclusion

The power of SQLAlchemy all boils down to understanding database
relationships and making use of the correct classes and methods. By leveraging
"convention over configuration", we're able to quickly set up complex
associations between multiple models with just a few lines of code.

The **one-to-many** and **many-to-many** relationships are the most common when
working with relational databases. You can apply the same concepts and code we
used in this lesson to any number of different domains, for example:

```txt
Driver -< Ride >- Passenger
Doctor -< Appointment >- Patient
Actor -< Character >- Movie
```

The code required to set up these relationships would look very similar to the
code we wrote in this lesson.

By understanding the conventions SQLAlchemy expects you to follow, and how
the underlying database relationships work, you have the ability to model all
kinds of complex, real-world concepts in your code!

***

## Resources

- [Python 3.8.13 Documentation](https://docs.python.org/3/)
- [SQLAlchemy ORM Documentation](https://docs.sqlalchemy.org/en/14/orm/)
- [Alembic 1.8.1 Documentation](https://alembic.sqlalchemy.org/en/latest/)
- [Basic Relationship Patterns - SQLAlchemy](https://docs.sqlalchemy.org/en/14/orm/basic_relationships.html)
