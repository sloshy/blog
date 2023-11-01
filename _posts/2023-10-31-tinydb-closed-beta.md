---
layout: post
title: TinyDB Closed Beta Announcement & Guide
---
Happy Halloween, everybody!
I have been a little behind on TinyDB, but never fear because I have completed an early preview build with some key improvements.

TinyDB, [as described previously](/2023/09/17/tinydb-update/), is an in-browser database application I've been working on in between trying to stay on top of the latest video games.
I had promised my early supporters that I would start early closed beta access some time in October, and to help reach that deadline I took a few days off to blitz to the finish line and reach an appropriate milestone.

If you are subscribed to my newsletter for TinyDB, you should have received an invite-only private link for trying TinyDB out as part of this closed beta.
At this link, you will be able to use a feature-limited version of the app that is similar to what was demoed at Scala Days Seattle 2023, with some changes and improvements.

## How To Use TinyDB (Closed Beta)
To use this version of TinyDB, there are two main "bars" at the top of your screen.
If you are on a mobile device, the first bar will collapse into a hamburger menu and is accessible with a button at the top-right of your screen.

To get started, enter a name for a database in the text field and press the "+" button to create a new table.
In addition, you may create a table as the result of a query, or through importing a CSV.
When importing a CSV, the first line will be treated as column header names.

If you encounter any difficulties, confusion, etc. please do not hesitate to email me directly.
If you are part of the closed beta, you can respond to my invite email or if you know me personally I will more than happily be receptive to feedback.

### Navigation
The nav bar has the following items:

* Import DB button, to load your saved DB JSON file
* New Table Name text box, for quick access to adding a table
* +, Query, and CSV buttons for adding a new table either as a new blank table, from a query, or from a CSV import
* Enums, to edit your set of custom enumerations
* A table select drop-down menu
* Export buttons

To quickly create a table, type a name into the text box and press `+`.
You can also import from a CSV or by querying your existing tables by pressing the buttons marked for query or CSV.

Switch between multiple tables by using the "selected table" drop-down menu.
When selecting a table, a secondary bar will appear under it with a way to easily rename the current table, turn it into a chart, or delete it.

To get started with a blank table, add new rows and columns with the "New Row" button at the bottom and the "+" button at the top-right of the edit area.
The blue buttons at the top of each column allow you to edit the name and type of the column.
The white buttons below will allow you to edit the values of that cel as text.
Clicking a "basic" type column cel will pop up a text box, whereas an "enum" type cel will show a drop-down menu of valid values.

To use enum columns, click the "Enum" button at the top of the navbar.
You can create a named enumeration here and add/delete values.
Then, you can edit a column for a table to set it as an enum, choose your newly created enum, and limit yourself to a known-good set of values.

In the future there will be a way to extract an enum from an existing column, so be on the lookout for that!

### Query Syntax
When constructing a query, certain operations are supported out of the box for this beta version, with more to come.
Queries are constructed by selecting certain parameters similar to a SQL query:

* `FROM` - The source table for this query
* `JOIN` - Another table that you would like to combine with the first table (optional)
* `AGGREGATE` - Zero or more "aggregate expressions" that can be used to group your results along certain axes
* `SELECT` - The columns you would like to keep from the resulting table
* `WHERE` - An expression to filter rows from the query

When you select a table with `FROM` and `JOIN`, you have the option to specify an alias.
This can help when referring to the tables by name later, which can be error-prone if your names are not short.

The set of columns you are allowed to refer to is listed below dynamically as you select tables for `FROM` and `JOIN`.
For example, a table `MyTable` with a column `Col1` will allow you to refer to `MyTable.Col1`, or if you specify the alias `A` for `MyTable`, that becomes `A.Col1`

Aggregation operations available are as follows:

* `COUNT <col>` - Count the number of times a column is not null
* `COUNT *` - Count the number of rows
* `SUM` - Sum the numerical values in this column
* `MIN` - Find the minimum value in this column
* `MAX` - Find the maximum value in this column
* `MEAN` - Find the average (mean) value in this column
* `MEDIAN` - Find the average (middle-most) value in this column
* `MODE` - Find the average (most common) value in this column

Aggregation syntax uses parenteses, so a valid aggregation expression might look like this:
```
MIN(A.Col1)
```

`SELECT` and `WHERE` statements work a little differently and have special syntax.
Unlike aggregations above, you will want to refer to multiple columns in `SELECT` or `WHERE` statements so you will use the `$` character to prefix column names.
For example, the column `Col1` on a table aliased `A` can be referred to as `$A.Col1` or `$"A.Col1"`, where quotes are used to help escape column names that use spaces or dollar signs.

A valid `SELECT` statement might be as follows:
```
$NumberCol - 4
```

The result of the expression will be all of the values of the column `NumberCol`, minus four.

The following operators are available for expressions:

* `+` - The ADDITION operator
* `-` - The SUBTRACTION operator
* `*` - The MULTIPLICATION operator
* `/` - The DIVISION operator
* `#` - The STRING LENGTH operator
* `!` - The NOT operator
* `&&` - The AND boolean operator
* `||` - The OR boolean operator
* `>` - The GREATER THAN numerical operator
* `>=` - The GREATER THAN OR EQUAL TO numerical operator
* `<` - The LESS THAN numerical operator
* `<=` - The LESS THAN OR EQUAL TO numerical operator
* `?` - The COALESCE operator (if the left-side value is null, replace it with the right-side value)

All of these expressions are usable in `SELECT` and `WHERE` statements alike.
So for example, you might want to select a column multiplied by four:

```
$NumCol * 4
```

Set an alias for it as `NewNumCol` to differentiate it from the original.
And then you might want to filter by which ones are greater than 36:

```
$NewNumCol > 36
```

### Loading, Saving, and Exporting
You can save your entire database to a JSON document with the `Database JSON` button under the `Export` section of the navbar.
You can later re-import it using the `Import DB` button on the left side near the application name.
Note that this method **does not save charts** at this time.
It will only save tables, including tables created from queries and CSV imports.

You may also export a table as a CSV by clicking the `Table CSV` button when on a valid table tab.

### Keyboard Shortcuts and Navigation
When editing a table, you can click the "+" button in the top-right of the editor section to add a new column, and as you are typing you may press "enter" or "tab" to move down and to the right, respectively.
The "shift" modifier key will go the opposite direction, much like you are editing cels in a spreadsheet program.