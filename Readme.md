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

## Specification

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