---
layout: post
title: "Adding Unique Constraints After the Fact in SQLAlchemy"
date: 2014-01-14 18:40
comments: true
published: false
categories: [python, SQLAlchemy, alembic]
---


In my [*****last post](link to last post), I wrote about writing a `get_one_or_create()` function for SQLAlchemy. Turns out that I had inadvertently created a race condition while horizontally scaling on Heroku. That post explains, in detail, the solution I came up with for moving forward, however I still had duplicate objects in my database that needed to remove. Since I'm on Heroku, I'm using Postgres (although I'm highly opinionated towards using Postgres often), so some of this may be Postgres specific, and I will do my best to note when that's the case.

I'll use the same example models that I used in my previous post. We had `Game` and `GameBeat` models:

<!-- more -->

```
class Game(db.Model):
    __tablename__  = 'games'
    id = db.Column(db.Integer, primary_key=True)
    provider_game_id = db.Column(db.String(255))
    provider_game_name = db.Column(db.String(255))
    provider_game_category = db.Column(db.String(255))

class GameBeat(db.Model):
    __tablename__ = 'game_beats'
    id = db.Column(db.Integer, primary_key=True)
    game_id = db.Column(db.Integer, db.ForeignKey('games.id'))
    game = db.relationship("Game",
                           backref="beats",
                           lazy="joined",
                           innerjoin=True)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'))
    user = db.relationship("User",
                           backref="game_beats",
                           lazy="joined",
                           innerjoin=True)
    beat_at = db.Column(db.DateTime())
```

but had accidentally created multiple `Game` objects with the same `provider_game_id` attribute. At the very least, we would rather have the class definition and database representation put a unique constraint on this attribute so that if we ever attempted to create such a duplicate `Game` object, we'll end up raising a `IntegrityError` and the duplicate won't actually get added to the datastore. (Again, my [*****previous post]() discusses how to handle this).

The class definition updates to:

```
class Game(db.Model):
    __tablename__  = 'games'
    id = db.Column(db.Integer, primary_key=True)
    provider_game_id = db.Column(db.String(255), unique=True)
    provider_game_name = db.Column(db.String(255))
    provider_game_category = db.Column(db.String(255))
```

We still, however need to make the update to our datastore (Postgres, in our case), along with finding and eliminating the duplicates we introduced into the system.

##Using `alembic` to update the datastore

If you're not using [`alembic`](http://alembic.readthedocs.org/en/latest/) with `SQLAlchemy`, you're doing it wrong. OK - that's a bit harsh (and if you're doing it some other way, let me know!), but if you ever plan to change your data model, you'll need to change your datastore along with it, and `alembic` is an excellent tool to help you along. It's version control for your database schema; think Git for your class models.

`alembic` provides the useful `autogenerate` option that can be used like so:

```
$ alembic --autogenerate -m "add unique contraint to Game.provider_game_id"
```

(Some [setup******](alembic autogenrate setup link) is required.) 

##Finding Duplicate Entries

As we're debugging the situation, one of the first things we'll want see how many duplicates we have of our object along the `provider_game_id`. I wrote this function do do just that:

```
from sqlalchemy import func

def n_dups(session, cls, attr):
    return session.query(getattr(cls,attr)).group_by(getattr(cls,attr)).\
            having(func.count(getattr(cls,attr)) > 1).count()
```

Running this, we'll use `n_dups(Game, 'provider_game_id')` and likely see something more than zero. All this tells us, however, is how many dups we have along that attribute. We now need to iterate though each duplicated `provider_game_id` and collect all the objects with said `provider_game_id`. Getting the duplicated `provider_game_ids` is similar to above:

```
def get_duplicated_attrs(session, cls, attr):
    return session.query(getattr(cls,attr)).group_by(getattr(cls,attr)).\
            having(func.count(getattr(cls,attr)) > 1).all()
```

So now we can get the duplicated objects:

```
def get_duplicated_objs(session, cls, attr, attr_value, order_by_attr):
    return session.query(cls).filter(getattr(cls,attr) == attr_value)).\
            order_by(order_by_attr).all()
```

The `order_by_attr` is going to come in handy in the next decision when we decide which duplicated objects get the axe.


##Eliminating Duplicate Entries

Our alembic migration file is also an excellent place to identify and eliminate any duplicates, for a few reasons:

1. If our database has duplicates on a row we are asking it to add a unique constraint to, we will get an `IntegrityError`, so we are required to do this before the command in the migration that adds the constraint is applied. Adding it here means that we always know it will be run before we try to migrate.
2. I generally don't like to clutter my applications with scripts that are (ideally) only run once, however I also do not want to run something like this in a REPL because you then loose the history of what was run. Adding it here means that we'll not only retain the history, but the exact point in which it was run.
3. It's already built into my build process (which is just a nice perk for me, but running `alembic` in your built process helps you to never forget running your migrations.)

When we delete the duplicate entries in our database, we have to decide on a rule for which one's to delete, and which *one* to keep. (You could also delete all, and then recreate them with your new protections against creating duplicates.) I decided to keep the oldest entry (not for any particularly good reason, except that I had originally created my objects with `created_at` attributes. Our example doesn't have these, so I'll use `id` which is effectively the same thing assuming the `id` is auto incremented.) Inside my alembic migration file, I added the following function

```
def remove_duplicates(session, cls, attr):
    duplicates = session.query(getattr(cls,attr)).group_by(getattr(cls,attr)).
                  having(func.count(getattr(cls, attr)) > 1).all()
    n_dups = len(duplicates)
    print '{}.{}: {} duplicates found'.format(cls.__name__, attr, n_dups)
    for duplicate in duplicates:
        objs = session.query(cls).
                filter(getattr(cls,attr) == getattr(duplicate, attr)).
                order_by('id').all()
        map(session.delete, objs[1:])
    session.commit()
    print '{}.{}: {} duplicates removed'.format(cls.__name__, attr, n_dups)
```

and within the `upgrade()` function I called this function for the objects and their attributes with duplicates:

```
def upgrade():
    remove_duplicates(db.session, Game, 'provider_game_id')
    op.create_unique_constraint(None,
                                "games",
                                ["provider_game_id"])
```

One thing you may run into at this point is an error complaining about 






