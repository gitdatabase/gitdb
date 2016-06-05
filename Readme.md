# GitDb 0.2 [DRAFT]

@author Bouke Versteegh

## Summary

Git has revolutionized collaborative software development. A similar revolution for collaborative database editing has not shown yet.
Succesfull crowd-sourcing projects exist, but are centralized, often closed source, may require permission to contribute, and do not provide 
methods of using the database locally, and especially provide no mechanism for branching, forking and version control.

GitDb is a standard for using a git repository as a database, to make collaborative editing of databases finally possible and easy.
The content and structure can be easily synchronized with existing databases such as Sql or Mongodb.


## Contents

- Introduction
- Goals
- Non-Goals
- GitDb Specification
    - Repository
    - Collections
    - Objects
    - References
    - (Collections of References)
    - (Lists of Primitives)
    - (Constraints)
- Synchronizing with other Databases

## Introduction

Git has proved to have revolutionized collaborative software development by allowing multiple users of the repository to make changes independent of each other,
while making it easy to import other user's changes back into the local repository. It is a decentralized model, where no-ones modifications can be forced upon another.
In collaborative software development, editors are free to use their own tools, and run the software locally without depending on third party services.

Today, there exists no such counterpart for collaborative database building. Existing collaborative database projects are centralized in nature, such as Wikipedia, Google Places, Pirate Bay.
Editors cannot freely use their own tools to edit add or remove data. This puts a limit to the speed with which the data can be improved. Especially small projects suffer from the fact that they are only editable through their website,
or they must build an api, for which other developers must build clients in turn. Additionally, everyone must work on a single agreed upon version of the database. For collaborative creativity, this is a problem, and has resulted in edit-wars on Wikipedia, where users cannot agree on which version of the record is preferable,
or many locked pages where special permission is required to even correct a spelling error.

In git based software development, these issues are solved by simply forking the project and start editing.
The owner of the base repository can decide freely whether to pull them in or not.

A naive solution such as using a database dump in a git repository, would not be a practical solution.
Databases are eventually used as backends for applications, so the dump would always need to be synchronized with a database server.
A dump overrides all of the database's contents on each import, creates a lot of unnecessary merge conflicts because records are dumped line by line,
and in git, modified lines that are adjecent create merge conflicts. Furthermore, this system cannot prevent duplicate primary keys, or misreferences due to id conflicts.
Finally it is forever tied to the database system used, and cannot readily be synchronized with other database systems (e.g. from mysql to mongodb).

GitDB provides a standard for storing database-like information to a git repository, in such a way that:

- it minimizes merge conflicts
- it lets users take full advantage of git, i.e. collaborative editing, forking, branching, pulling, merging, gitignore
- it is easily synchronized in two ways with multiple existing database systems
- application developers can still use their current database system, and use GitDb to enable collaboration and versioning

## Goals

- provide a standard for storing database contents and structure to a git repository
- any project synchronizing their database with a GitDB repository can directly benefit from crowd-sourced contributions, version control, and tools that are available for Git and GitDb
- setting up synchronization should require minimal effort
- minimize number of merge conflicts for crud and structural changes
- make the barriers to use GitDB as low as possible
- should be compatible (synchronizable) with any popular database systems
- adherence to the gitdb standard should ensure that the database is convertible to any database system for which drivers are available
- enhancing an existing database with GitDb does not require high changes to the database structure

## Non-Goals

- using GitDb as a live backend for high performance applications
- support storing large amounts of machine generated data
- provide a mechanism for querying the GitDB repository
- optimize for space efficiency
- standardize how a GitDb repository should be converted to a specific database platform.  
  Examples and algorithms will be provided to facilitate the development of drivers.

## GitDb Specification

### Repository

A GitDb repository is a Git repository containing one or more collections of objects.

A repository must include the following empty file:

    /.gitdb

Restrictions:
- The repository may not contain any directories other than the specified collections
- The root of the repository may contain other files, these are not part of the database

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
- Collections may not contain any files

### Objects

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
- Objects may not be moved to another collection [1]
- Objects that were deleted may be restored with their original name [2]


1. To discuss. An application storing users a collection of owned items, may need to move an owned item to another user.
   Either this restriction should be removed, or specified more precisely to enable this use case.

### Attributes

Attributes are name-value pairs within objects. Attributes are stored differently depending on their value type.

These are the available value types:

- Primitives (``string``, ``number``, ``true``, ``false``)
- Dictionary
- Collection
- Reference

Restrictions:

- Attribute names must be unique within their parent directory (object or dictionary)
- Attribute name equality tests must be case insensitive

#### Primitives

Available primitive value types are: ``string``, ``number``, ``true``, ``false``. These correspond to the same value types in JSON.

Primitives are stored as files.

A repository with a user that has a name (type ``string``), might contain the following file:

    /users/0130f996-3930-45b7-9413-b5582f318fda/name

Primitive values are stored in files encoded as JSON values.

For example, if the user's name was ``Óscar González`` the contents of the file would be:

    "\u00d3scar Gonz\u00e1lez"

Restrictions:

- The whole content of the file must be a single parsable JSON value
- The file must not be terminated by a newline
- The file must not be empty
- ``null`` is not a valid value. Undefined values must be modeled by omitting the attribute completely.

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

#### Lists of primitives

In this version of GitDb, lists of primitive values are not supported.

This aspect is open for discussion.

A few points to consider:
- Since primite values have no attributes, their order should be stored somehow
- How to identify an item on the list? If [1,2,3] was changed to [1,3], was 2 removed, or changed to 3 and 3 removed? The synchronization of these modification to MySQL may not be trivial. Suggestions welcome.

Against supporting lists:
- MySQL does not support multiple values per column

One method of modeling lists could be:
- An attribute stored as a file, where each line is a json encoded value.

Ordered lists make collaboration difficult, since the values do not have identifiers.


#### Constraints

The specification has not defined how to model constraints yet.

# Synchronizing with other databases

GitDb does not specify how a repository should be mapped to a database of another type, however, this section is provided as a helpful guide in developing drivers for this purpose.
Suggestions in this section are not part of the specification.

The following list indicates how a GitDb repository may correspond to a database.

- collection — table, collection
- object — record, document
- attribute — column, property
- reference — foreign key


As an example, the following repository could be mapped to MySQL as follows.

**GitDb Repository**

```
/countries/384fbe/name (contents: "USA")
/users/1f34a8/name (contents: "John")
/users/1f34a8/country (link: ../../countries/384fbe)
```
_Note: Object names are shortened for readability._

**MySQL**

```
# countries
id | uuid   | name
1  | 384fbe | USA
```

```
# users
id | uuid   | name | country
1  | 1f34a8 | John | 2
```
_Note: This particular mapping does not use the object names as primary keys, which makes sense for performance reasons.
