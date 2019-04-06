---
layout: guide
title: Rest.li Snapshots and Resource Compatibility Checking
permalink: /modeling/compatibility_check
index: 2
excerpt: Due to the fact that resources and clients can be upgraded separately, it is very important to developers that they be notified of any changes they make that could cause backwards or forwards compatibility issues. To that end, Rest.li uses a form of expanded IDLs, called Snapshots, to keep track of the state of resources and check compatibility between resource iterations.
---
# Rest.li Snapshots and Resource Compatibility Checking

## Contents

* [Introduction](#introduction)
* [Running the Compatibility Checker](#running-the-compatibility-checker)
* [Compatibility Levels](#compatibility-levels)
* [Why is Adding to an Enum Considered Backwards Incompatible?](#why-is-adding-to-an-enum-considered-backwards-incompatible)
* [Error Messages](#error-messages)
* [Location of Snapshots/IDLs](#location-of-snapshotsidls)
* [The Compatibility Checker and IDLs](#the-compatibility-checker-and-idls)
* [Continuous Integration Environments](#continuous-integration-environments)

## Introduction

Due to the fact that resources and clients can be upgraded separately, it is very important to developers that they be notified of any changes they make that could cause backwards or forwards compatibility issues. To that end, Rest.li uses a form of expanded [IDLs](http://en.wikipedia.org/wiki/Interface_description_language), called Snapshots, to keep track of the state of resources and check compatibility between resource iterations.

## Running the Compatibility Checker

The Snapshot Compatibility checker will be automatically run during a basic gradle build. However, if you wish to run the compatibility checker as a stand-alone on a particular target, you can do so by running `gradle :[target]:checkRestModel`. Additionally, there are four compatibility levels that the checker can be run on, by adding the argument `-Prest.model.compatibility=[compatibility-level]` to the `gradle` command.

The compatibility checker generates `.snapshot.json` files. 

## Compatibility Levels

There are four levels of compatibility. From least permissive to most, they are:

**equivalent** - If the check is run in equivalent mode, no changes to Resources or PDSCs will pass.

**backwards** - Changes that are considered backwards compatible will pass, otherwise, changes will fail.

**ignore** - The compatibility checker is run, but all changes will pass. All changes, backwards compatible and backwards incompatible, will be printed.

**off** - The compatibility checker will not be run at all.

By default, the compatibility checker will be run on backwards compatibility mode. There are two ways to change the mode the compatibility checker is run on. For one, you can add a flag to the gradle build itself: `--Prest.model.compatibility=<compatLevel>`

Alternately, if you want to change the default compatibility level for all builds on a machine, you can create or edit the file `~/.gradle/gradle.properties` to contain the line `rest.model.compatibility=<compatLevel>`

## Why is Adding to an Enum Considered Backwards Incompatible?

Many developers are surprised that adding to an enum is considered a backwards incompatible change.

But, while Rest.li is designed with features to make it easier to add symbols to enums, it cannot possibly guarantee that adding a enum symbols is backward compatible.

To make it easier to add values in the enum data schema, java enum classes generated by Rest.li that correspond to enum data schema always contains a special "$UKNOWN" symbol. Whenever Rest.li deserializes enum data that contains a symbol that is not present in the java enum, Rest.li maps it to "$UNKNOWN". When the enum is accessed via accessor implemented by a data template, the accessor will return the new symbol as the java "$UNKNOWN" symbol. This gives readers of the enum the opportunity to check if the enum is "$UNKNOWN", and if it is, handle is in the best possible way.

However, it's still not possible to guarantee backward compatibility, even with the "$UKNOWN" symbol available.  It's possible that clients did not handle the "$UKNOWN" symbol in the best possible way,  and even if they did it may be that they cannot do anything other than fail if they encounter a enum symbol they do not recognize.   In many practical applications,  it is not feasible to assess how all clients have been coded to handle new enum symbols, particularly when there are many clients.  In such cases, adding a new enum symbol might break a unknown number of clients.

It's true that there may be well controlled use case were an enum is used only by a single client and server that are maintained by the same developers.   In these very specific cases, it may be easy for the developers to know that new enum symbols can be safely added at any time.  But if additional clients might be added in the future,  it is still risky to get in the habit of adding enum symbols "as-if" they are backward compatible changes. Once additional clients start using the API,  adding a symbol could break them.

Given all these potential issues with adding a enum symbol, it's important to think of adding enum symbols as backward incompatible.  If a new symbols is to be added, a migration strategy for adding the enum symbol(s) must be performed just as for any other backward incompatible change.  Note that this is only possible when all clients are known and it is possible to coordinate changes with them.  If this is not the case,  one should consider making a backward compatible change (such as adding a new optional field containing a new enum field with more symbols) and supporting the existing clients, with the existing enum symbols, indefinitely.

One possible backwards incompatible migration strategy for adding a enum symbol might be:
* Add the symbol as a backward incompatible change to the data schema,  use the rest.li [gradle `rest.model.compatibility` flag](/rest.li/setup/gradle#compatibility) to "ignore" the backward incompatible change.  If you are using [semantic versioning](http://semver.org/), you should also increment your MAJOR version number.  Do NOT start writing data that contains the new enum value yet.
* Once the backward incompatible change has been published, REST API clients must be notified to the change. They should be provided with details on the enum symbol that has been added and how to migrate their applications.   Clients should NOT write using the new symbol yet (the resource implementation may choose to reject requests to POST or PUT data with the new symbol).
* Only once all clients have migrated to the new (backward incompatible version) of the API, clients and the resource implementation may start writing data using the new symbol.

Some things to watch out for when adding enum symbols:
* Rest.li provides schema and data translation to avro.  While unknown symbols can be deserialized by older rest.li consumers (because rest.li does not require the schema to de-serialize), it doesn't work for data persisted as Avro.  Any attempt to deserialize an avro record containing the new enumeration value with an older schema lacking that enum will fail.
It is safe to ignore the incompatibility message if you are sure this will not happen, or you can work with your clients to make sure that it doesn't. In general, we only provide notifications about backwards incompatible changes as a tool for the developer.  You are always free to ignore backwards-incompatible change messages if you know that the change will not cause problems, or are willing to take steps to ensure that it will not.

## Error Messages

There are a number of error messages that can appear during compatibility checking. They will look something like this:

```
idl compatibility report between published "/publishedlocation/com.namespace.resource.snapshot.json" and current "/currentlocation/com.namespace.resource.snapshot.json":
  Incompatible changes:
    1) /location : detailed error message
```

This will tell you what resource is causing the problem, whether the change is backwards incompatible or not, and where exactly in the resource the problem is. You can then either use this information to fix the problem, or ignore it and re-run the build with a compatibility level that will cause it to pass.

## Location of Snapshots/IDLs

The canonical snapshots for a set of resources will be located in the api project associated with those resources. If this project is located in some directory project-api, then the snapshots will be located in `project-api/src/main/snapshot`. Similarly, canonical IDLs will be located in `project-api/src/main/idl`. Though the files in these directories are generated, we strongly suggest that they be checked in. The IDLs must be checked in to correctly generate builders, and checking in the snapshots ensures that developers will receive accurate compatibility messages when making changes.

The temporary snapshots for a set of resources will be located within the same project as the java Resource files themselves. If the project for the Resource file is in a directory `project-impl`, then the snapshots will be located in `project-impl/src/mainGeneratedRest/snapshot`. Again, similarly, the temporary IDLs will be located in `project-impl/src/mainGeneratedRest/idl`. Unlike the canonical files, these generated files should **NOT** be checked in.

## The Compatibility Checker and IDLs

The compatibility checker will also do limited IDL checks. By default, IDLs will only be checked to make sure there are no orphan IDLs from newly removed Resources and no missing IDLs from newly added Resources. A compatibility message may be printed saying that a resource has been added or removed. If a resource has been removed, you will need to remove the canonical IDL yourself. (This is also true for Snapshots).

## Continuous Integration Environments

If you are running a continuous integration environment on a Rest.li project, you will want to run your compatibility checker on `equivalent`. This will prevent your canonical IDLs and Snapshots from getting out of sync with the Resources they represent. You can do this either by running each build with the flag `-Prest.model.compatibility=equivalent`, or by creating or editing the `~/.gradle/gradle.properties` file to contain the line `rest.model.compatibility=equivalent`