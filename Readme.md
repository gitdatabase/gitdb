# Notes

- GitDB should be a standard for storing database-like information to a git repository.
- It does not tell how it should be mapped/imported exported to SQL/Mongo etc.
- Makers of tools to synchronize with gitdb should decide by themselves how to do that
- Example drivers will be provided for a few of the most popular database systems.
- The repository does not contain information on the mapping.


# GitDb 0.1

## Summary

A standard for importing and exporting data from a database to a git repository, to facilitate collaborative editing of databases.

## Introduction

*todo*

Main points:
- collaborative databases right now are centralized (wikipedia, google places)
- reading the data requires access to a centralized website, when this website is offline, the database is inaccessible (pirate bay)
- writing to the database requires using the centralized tools (wikipedia), no freedom to use own tools, or only available through an api.

## Goals
- provide an intuitive standard for converting a database structure and content to a git repository, and vice-versa
- minimize number of merge conflicts
- be agnostic about the corresponding database
- support relational databases with foreign key constraints
- support document oriented databases
- allow databases to use their own native primary keys
- allow two users to add a record to a table without creating a duplicate primary key, and without requiring manual merge
- a merge conflict never requires changing of record ids. record ids are permanent
- a git-database can easily be checked for consistency

## Content

- Specification
    - Collections
    - Objects
        - Attributes
            - Primitives
            - Dictionary
            - Collection
    - Relations
    - Constraints

## Specification

### Collections

In gitdb data is organized in collections, which is simply a directory in the root of the repository.

For example, an application keeping track of users and their posts contains the following directories:

    /users
    /posts

Restrictions:
- Collection names must match the following regex pattern: [a-zA-Z0-9_]+
- Collection name equality tests must be case insensitive
- Collection names must be unique within the repository
- Collections must contain zero or more objects

### Objects:

Objects are stored within collections. An object is represented by a sub-directory of a collection, with a UUID-4 name.

For example, a database containing one user and two posts has the following directories:

    /users/0130f996-3930-45b7-9413-b5582f318fda
    /posts/7403057b-f183-45db-bcce-f05f509fc3d7
    /posts/8e7eed2b-a285-44e5-b69e-b9961548b64f

Restrictions:
- Object names must be a valid UUID-4 identifier
- Object names must be lowercase
- Object names must be unique within the _repository_
- Object names may not be changed
- Objects may be deleted
- Objects may be added
- Objects may not be moved to another collection
- Objects that were deleted may be restored with their original name [? to discuss]

### Attributes:

Attributes are name-value pairs within objects. Attributes are stored differently depending on their value type.

These are the available value types:
- Primitives (``string``, ``number``, ``true``, ``false``, ``null``)
- Dictionary
- Collection
- Reference

Restrictions:
- Attribute name equality tests must be case insensitive
- Attribute names must be unique within their parent directory (object or dictionary)

#### Primitives

Available primitive value types are: ``string``, ``number``, ``true``, ``false``, ``null``. These correspond to the same value types in JSON.

Attributes of these types are stored as files.

A repository with a user that has a name (type ``string``), might contain the following file:

    /users/0130f996-3930-45b7-9413-b5582f318fda/name

Primitive values are stored in files encoded as JSON values.

For example, if the user's name was ``Óscar González`` the contents of the file would be:

    "\u00d3scar Gonz\u00e1lez"

Value restrictions:
- A value must be a primitive. I.e: it must not be an array or object
- A value may be null.

File content restrictions:
- The whole content of the file must be a single parsable JSON value
- The file must not be terminated by a newline
- The file must not be empty

#### Dictionary

A dictionary is a named group of attributes, stored as a directory within an object or another dictionary.

For example, a repository with a user that has an ``address`` with three attributes ``street``, ``city``, ``country`` might contain the following files.

    /users/0130f996-3930-45b7-9413-b5582f318fda/address/street
    /users/0130f996-3930-45b7-9413-b5582f318fda/address/city
    /users/0130f996-3930-45b7-9413-b5582f318fda/address/country

#### Collection

An object or a dictionary can contain a named collection of objects. It is stored exactly like a root level collection,
within the parent object or dictionary.

For example, a social networking application with one user and two posts (with one attribute ``content``) may contain the following files directories.

    /users/0130f996-3930-45b7-9413-b5582f318fda/posts/413f226d-34e2-41fa-a79f-5e9ad38a098a/content
    /users/0130f996-3930-45b7-9413-b5582f318fda/posts/83a42bfb-3fc1-4a4a-857e-abfe0f7ad752/content

Restrictions:
- The order of a collection is not defined. Ordering objects within a collection must be implemented at the application level, for example by storing an ``index`` attribute within each object, with a value representing its position in within the collection.
- A collection must not be empty. An empty collection must be modeled by omiting the directory.
- Otherwise, the same restrictions to root-level collections apply.


#### Reference

A reference is an attribute whose value is another object within the repository. It is stored as a relative symbolic link to the object's directory.

For example, a user who lives in the United States, could be modeled in this way:

    /countries/aeb2070a-4ad8-4cba-892b-aa6548aceace
    /users/8bebfeba-c272-483d-831e-91a184bf88ab/country
        --> ../../countries/aeb2070a-4ad8-4cba-892b-aa6548aceace

As another example, an application where each user has a collection of posts, and liked posts, could be modeled as follows:

    /users/0130f996-3930-45b7-9413-b5582f318fda/posts/413f226d-34e2-41fa-a79f-5e9ad38a098a/content
    /users/8bebfeba-c272-483d-831e-91a184bf88ab/likedPosts/86e6c742-c2d0-4152-b6a8-17e0e4f62589/post
        --> ../../0130f996-3930-45b7-9413-b5582f318fda/posts/413f226d-34e2-41fa-a79f-5e9ad38a098a

Note that each likedPost is an object with its own unique name, containing an attribute ``post``, which references the post object.

#### Collections of references

In this version of GitDB, collections of references are not supported.

The following example is invalid, because the file (``likedPosts/86e...589``) is itself a reference,
while collections must only contain objects, and may not contain attributes or values directly.

    /users/8bebfeba-c272-483d-831e-91a184bf88ab/likedPosts/86e6c742-c2d0-4152-b6a8-17e0e4f62589
        --> ../../0130f996-3930-45b7-9413-b5582f318fda/posts/413f226d-34e2-41fa-a79f-5e9ad38a098a


## RDBS

Relational Database Systems

Mapping a relational database to the filesystem

```
/TABLE_NAME/RECORD_ID/COLUMN_NAME
```

The value of the column is stored in the file ``COLUMN_NAME``, encoded in JSON.

**Foreign keys**

A foreign key is simply a column whose value is a symlink to the referenced record directory.

**Example**

```
Table: users
----------------------------
id | first_name | last_name
----------------------------
 1 | Alice      | Jones
 2 | Bob        | Smith
 
Table: messages
----------------------------------------------
id | from_user_id | to_user_id | body
----------------------------------------------
 1 |            1 |          2 | Hi Bob
 2 |            2 |          1 | Hello Alice
```

List of files in git working directory.

```
/.gitignore
./users/18da2b15-1708-4cec-a583-564e79c9bddb/id
./users/18da2b15-1708-4cec-a583-564e79c9bddb/first_name
./users/18da2b15-1708-4cec-a583-564e79c9bddb/last_name
./users/2bc4b0fa-e6ba-4866-a6ae-be92a1eebd64/id
./users/2bc4b0fa-e6ba-4866-a6ae-be92a1eebd64/first_name
./users/2bc4b0fa-e6ba-4866-a6ae-be92a1eebd64/last_name
./messages/6de01425-8a55-4f2f-959e-595aa832d5c2/id
./messages/6de01425-8a55-4f2f-959e-595aa832d5c2/from_user_id --> ../../users/18da2b15-1708-4cec-a583-564e79c9bddb
./messages/6de01425-8a55-4f2f-959e-595aa832d5c2/to_user_id --> ../../users/2bc4b0fa-e6ba-4866-a6ae-be92a1eebd64
./messages/6de01425-8a55-4f2f-959e-595aa832d5c2/message
./messages/ddaa555a-6a11-4e05-bdd0-47c0ab2a5f1a/id
./messages/ddaa555a-6a11-4e05-bdd0-47c0ab2a5f1a/from_user_id --> ../../users/2bc4b0fa-e6ba-4866-a6ae-be92a1eebd64
./messages/ddaa555a-6a11-4e05-bdd0-47c0ab2a5f1a/to_user_id --> ../../users/18da2b15-1708-4cec-a583-564e79c9bddb
./messages/ddaa555a-6a11-4e05-bdd0-47c0ab2a5f1a/message
```

Contents of ``.gitignore``

```
/users/*/id
/messages/*/id
```

### UUIDS vs Primary Keys

The primary keys in the table are not used to identify records within the git repository, as this would raise conflicts when multiple participants use the same subsequent id to create a record. Therefor, every record gets a universally unique id.

- Todo: How to map the ID to UUID and vice-versa.
  - store UUID in table
    - con: need to modify tables to use gitdb
  - store id in repository (gitignored)
    - con: need a local checkout of the repository, while a bare .git repository would be more lightweight
  - keep the mapping somewhere else. Perhaps inside the .git directory.
    - con: not very intuitive

- Questions to ansnwer:
  - what happens when you switch branches. is the mapping still valid? can we ensure that records in the database get the same UUID?



## Document Database
```
COLLECTION_NAME:
  RECORD_ID
    PROPERTY_NAME: PROPERTY_VALUE
    or
    PROPERTY_NAME:
      PROPERTY_NAME: PROPERTY_VALUE
```

Example

```
users:
  18da2b15-1708-4cec-a583-564e79c9bddb:
    first_name: "Alice"
    last_name: "Jones"
    address:
      home:
        street: "Foobar boulevard"
        house_number: 1
        
```