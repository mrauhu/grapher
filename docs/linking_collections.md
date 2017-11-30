# Linking Collections

Let's learn what types of links we can do between collections, and what is the best way to do them.

First, we begin with an illustration of the power of Grapher:

Let's assume our posts, contain a field, called `authorId` which represents an actual `_id` from `Meteor.users`,
if you wanted to get the post and the author's name, you had to first fetch the post,
and then get the author's name based on his `_id`.

Something like:
```js
Meteor.methods({
    getPost({postId}) {
        let post = Posts.findOne(postId, {
            fields: {
                title: 1,
                createdAt: 1,
                authorId: 1,
            }
        });
        
        if (!post) { throw new Meteor.Error('not-found') }
        const author = Meteor.users.findOne(post.authorId, {
            fields: {
                firstName: 1,
                lastName: 1
            }
        });
        
        Object.assign(post, {author});
        
        return post;
    }
})
```

With Grapher, your code above is transformed to:

```js
Meteor.methods({
    getPost({postId}) {
        let post = Posts.createQuery({
            $filters: {_id: postId},
            title: 1,
            createdAt: 1,
            author: {
                firstName: 1,
                lastName: 1
            }
        });
        
        return post.fetchOne();
    }
})
```

This is just a simple illustration, imagine the scenario, in which you had comments,
and the comments had authors, and you needed their avatar. Your code can easily
grow to many lines of code, and it will be much less performant because of the many round-trips you do to
the database.

## Linking Types

To make the link illustrated in the example above, we create a separate `links.js` file that is 
imported separately outside the collection module. We'll understand later why.

```js
// file: /imports/db/posts/links.js
import Posts from '...';

Posts.addLinks({
    'author': {
        type: 'one',
        collection: Meteor.users,
        field: 'authorId',
    }
})
```

That was it. You created the link, and now you can use the query illustrated above. 
We decided to choose `author` as a name for our link and `authorId` the field to store it in, but any string will do.

### Inversed Links

Because we linked `Posts` with `Meteor.users` it means that we can also get `posts` if we are in a User.
But because the link is stored in `Posts` we need a new type of linking, and we call it `Inversed Link`

```js
// file: /imports/db/users/links.js
import Posts from '...';

Meteor.users.addLinks({
    'posts': {
        collection: Posts,
        inversedBy: 'author'
    }
})
```

`author` represents the link name that was defined inside Posts. By defining inversed links we can do:

```js
Meteor.users.createQuery({
    posts: {
        title: 1
    }
})
```

### One and Many

Above you've noticed a `type: 'one'` in the link definition, but let's say we have a `Post` that belongs to many `Categories`,
which have their own collection into the database. This means that we need to relate with more than a single element.


```js
// file: /imports/db/posts/links.js
import Posts from '...';
import Categories from '...';

Posts.addLinks({
    'author': { ... },
    'categories': {
        type: 'many',
        collection: Categories,
        field: 'categoryIds',
    }
})
```

In this case, `categoryIds` is an array of Strings, each String, representing `_id` from `Categories` collection.

And, ofcourse, you can also create an inversed link from `Categories`, so you can use it inside the `query`
```js
// file: /imports/db/posts/links.js
import Categories from '...';
import Posts from '...';

Categories.addLinks({
    'posts': {
        collection: Posts,
        inversedBy: 'categories'
    }
})
```

## Meta Links

We use a `meta` link when we want to add additional data about the relationship. For example,
a user can belong in a single `Group`, but we need to know when he joined that group and what roles he has in it.

```js
// file: /imports/db/users/links.js
import Groups from '...'

Meteor.users.addLinks({
    group: {
        type: 'one',
        collection: Groups,
        field: 'groupLink',
        metadata: true,
    }
})
```

Notice the new option `metadata: true` this means that `groupLink` is no longer a `String`, but an `Object` that looks like this:

```
// inside a Meteor.users document
{
    ...
    groupLink: {
        _id: 'XXX',
        roles: 'ADMIN',
        createdAt: Date,
    }
}
```

Let's see how this works out in our query:

```js
const user = Meteor.users.createQuery({
    $filters: {_id: userId},
    group: {
        name: 1,
    }
}).fetchOne()
```

`user` will look like this:

```
{
    _id: userId,
    group: {
        $metadata: {
            roles: 'ADMIN',
            createdAt: Date
        },
        name: 'My Funky Group'
    }
}
```

We store the metadata of the link inside a special `$metadata` field. And this works from inversed side as well:

```js
Groups.addLinks({
    users: {
        collection: Meteor.users,
        inversedBy: 'group'
    }
});

const group = Groups.createQuery({
    $filters: {_id: groupId},
    name: 1,
    users: {
        firstName: 1,
    }
}).fetchOne()
```

`group` will look like:
```
{
    _id: groupId,
    name: 'My Funky Group',
    users: [
        {
            $metadata: {
                roles: 'ADMIN',
                createdAt: Date
            },
            _id: userId,
            firstName: 'My Funky FirstName',
        }
    ]
}
```

The same principles apply to `meta` links that are `type: 'many'`, if we change that in the example above. 
The storage field will look like:

```
{
    groupLinks: [
        {_id: 'groupId', roles: 'ADMIN', createdAt: Date},
        ...
    ]
}
```

And they work the same as you expect, from the inversed side as well.

I know what question comes to your mind right now, what if I want to put a field inside the metadata,
I want to store who added that user (`addedBy`) to that link and fetch it, how do we do ?

Currently, this is not possible, because it has deep implications with how the Hypernova Module works,
but you can achieve this in a very performant way, by abstracting it into a separate collection:

```js
// file: /imports/db/groupUserLinks/links.js
import Groups from '...';
import GroupUserLinks from '...';

GroupUserLinks.addLinks({
    user: {
        type: 'one',
        collection: Meteor.users,
        field: 'userId'
    },
    adder: {
        type: 'one',
        collection: Meteor.users,
        field: 'addedBy'
    },
    group: {
        type: 'one',
        collection: Meteor.users,
        field: 'groupId'
    }
})

// file: /imports/db/users/links.js
Meteor.users.addLinks({
    groupLink: {
        collection: GroupUserLinks,
        type: 'one',
        field: 'groupLinkId',
    }
})
```

And the query will look like this:
```js
Meteor.users.createQuery({
    groupLink: {
        group: {
            name: 1,
        },
        adder: {
            firstName: 1
        },
        roles: 1,
        createdAt: 1,
    }
})
```

## Link Loopback

No one stops you from linking a collection to itself, say you have a list of friends which are also users:
```js
Meteor.users.addLinks({
    friends: {
        collection: Meteor.users,
        type: 'many',
        field: 'friendIds',
    }
});
```

Say you want to get your friends, and friends of friends, and friends of friends of friends!
```js
Meteor.users.createQuery({
    $filters: {_id: userId},
    friends: {
        nickname: 1,
        friends: {
            nickname: 1,
            friends: {
                nickname: 1,
            }
        }
    } 
});
```

## Uniqueness

The `type: 'one'` doesn't necessarily guarantee uniqueness from the inversed side. 
For example, if we have inside `Comments` a link with `Posts` with `type: 'one'` inside postId,
it does not mean that when I fetch the posts with comments, comments will be an array.

But if you want to have a one to one relationship, and you want grapher to give you an object,
instead of an array you can do so:

```js
Meteor.users.addLinks({
    paymentProfile: {
        collection: PaymentProfiles,
        inversedBy: 'user'
    }
});

PaymentProfiles.addLinks({
    user: {
        field: 'userId',
        collection: Meteor.users,
        type: 'one',
        unique: true
    }
})
```

Now fetching:
```js
Meteor.users.createQuery({
    paymentProfile: {
        type: 1,
        last4digits: 1,
    } 
});
```

`paymentProfile` inside `user` will be an object because it knows it should be unique.

## Data Consistency

This is referring to having consistency amongst links. 

Let's say I have a `thread` with multiple `members` from `Meteor.users`. If a `user` is deleted from the database, we don't want to keep unexisting references. 
So after we delete a user, all threads containing that users should be cleaned.

This is done automatically by Grapher so you don't have to deal with it.
The only rule is that `Meteor.users` collection needs to have an inversed link to `Threads`.

In conclusion, if you want to benefit of this, you have to define inversed links for every direct links.

## Autoremoval

```js
Meteor.users.addLinks({
    'posts': {
        collection: Posts,
        inversedBy: 'author',
        autoremove: true
    }
});
```

After you deleted a user, all the links that have `autoremove: true` will be deleted.

This works from the `direct` side as well, not only from `inversed` side.

## Indexing

As a rule of thumb, you must index all of your links. Because that's how you achieve absolute performance.

This is not done by default, to allow the developer flexibility, but you can do it simply enough from the direct side definition of the link:

```js
PaymentProfiles.addLinks({
    user: {
        field: 'userId',
        collection: Meteor.users,
        type: 'one',
        unique: true,
        index: true,
    }
})
```

The index is applied only on the `_id`, meaning that if you have `meta` links, other fields present in that object will not be indexed.

If you have `unique: true` set, the index will also apply a unique constraint to it.

## Top Level Fields

Grapher currently supports only top level fields for storing data. One of the reasons it doesn't allow nested fields
is to enforce the developer to think relational and eliminate large and complex documents by abstracting them into collections.

In the future, this limitation may change, but for now you can work around this and keep your code elegant.

## Conclusion

Using these simple techniques, you can create a beautiful database schemas inside MongoDB that are relational and very simple to fetch,
you will eliminate almost all your boilerplate code around this and allows you to focus on more important things.

On top of a cleaner code you benefit from the `Hypernova Module` which minimizes the database requests to a
predictable number (no. of collection nodes). 








