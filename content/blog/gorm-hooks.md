---
title: "Using GORM Hooks to Clean up Test Fixtures in Golang"
date: 2019-01-27T18:05:06+05:30
tags: ["Golang", "Go", "Gorm", "Software Testing"]
slug: "test cleanup with gorm hooks"
---

![](/gopher.jpeg)

If you’ve ever written code in Golang that interfaces with the database, chances are that you already know [GORM](https://github.com/jinzhu/gorm). With GORM, creating, updating, deleting records is super simple.

<script src="https://gist.github.com/jarifibrahim/72c7b24f8ba08baa1a2266c302a8c625.js"></script>

But GORM offers a lot more than just basic database operations. One of my favourites is the ability to attach database [Hooks](http://gorm.io/docs/hooks.html). Database hooks can be used to do all kinds of cool stuff like automatically deleting records created by test fixtures, logging information about when a record was inserted/deleted, updating records in one table when another is changed and so on.

## Deleting Records Created by Test Fixtures with GORM
A common practice is to create test prerequisites via test fixtures. Let’s say you were testing an application which requires some data to be present in the database. The usual way of testing this would be to insert some test data in the database and then test the application. It is extremely important that the database is in a clean state between test runs and there are no leftovers in the database. You would never want the left over from the previous test affect your current test.

But there’s a problem — How do you remove all the records created by the test fixtures? You could drop the entire database but let’s assume you don’t want to do that. So how do you keep your test environment in a clean and consistent state?

One easy (not really!) and not-so-cool way of doing that would be to manually delete every record that you inserted in the database.

<script src="https://gist.github.com/jarifibrahim/3650b2dc27a292008efd43c3b346c63a.js"></script>

## So how you do delete records elegantly? GORM Hooks to the Rescue!

The easiest way is to set up a `onCreate` hook on the database and every time you create a record you store its information in a temporary data structure. Once your test completes, you can delete the all the records listed in the temporary data structure. The following code shows a simple function that can be used to remove all records created in the database after the hook was set up.

<script src="https://gist.github.com/jarifibrahim/039090459ff04e14233dccd52ee2d9cf.js"></script>

The `entries` array in the above snippet stores the list of records created after the hook was set up and these records would automatically be deleted when the returned function is called.

The following code snippet shows a more thorough example

<script src="https://gist.github.com/jarifibrahim/6f800a33079722653745ca6fd2f96772.js"></script>

The test in the above code snippet creates 3 records in the database and the GORM hook stores those records in the `entities` array. Once the test is done, all the records are automatically deleted from the database. The following code snippet shows the output of the test in the code snippet above

```
➜ (foo) go test -v
=== RUN   TestCleanup
Inserted entities of products with id=40
Inserted entities of products with id=41
Inserted entities of products with id=42
Deleting entities from 'products' table with key 42
Deleting entities from 'products' table with key 41
Deleting entities from 'products' table with key 40
2019/01/27 15:12:12 [info] removing callback `cleanupHook` from /home/ijarif/Projects/go/src/github.com/jarifibrahim/foo/cleanup_test.go:81
--- PASS: TestCleanup (0.02s)
PASS
ok   github.com/jarifibrahim/foo 0.022s
```

This is how you can ensure your test environment remains in a clean and consistent state across test runs. If you’d like to read more about GORM hooks, head over to http://gorm.io/docs/hooks.html

