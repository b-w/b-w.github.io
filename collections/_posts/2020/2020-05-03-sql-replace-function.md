---
layout: post
title: "A pure T-SQL replace-all function using PATINDEX"
categories: blog
---

Although SQL Server is a powerful and fully-featured database engine, one of the things it has always lacked is support for regular expressions. This isn't a super big deal, because you don't (or at least shouldn't) need regexes in your database all that often, and if you do need them you can always write a trivial C# extension for it thanks to the SQL CLR.

Or, well, at least you can if you're running SQL Server on prem. In Azure, it's another story. In the cloud, the CLR is only available if you're running a managed instance; it's not available if you're running single databases or elastic pools.

Without the CLR, you're basically out of luck as far as regexes go. But, you can hack something together using [PATINDEX](https://docs.microsoft.com/en-us/sql/t-sql/functions/patindex-transact-sql). In this post, I'll show how PATINDEX works and how you can use it to write a poor man's pattern replace function.

## Index that pattern

PATINDEX is a built-in function in SQL Server. It finds the first index in a string that matches a given pattern. It doesn't support regex; rather it supports the same patterns as the [LIKE](https://docs.microsoft.com/en-us/sql/t-sql/language-elements/like-transact-sql#pattern-matching-by-using-like) operator. That means it can do some limited wildcard pattern matching.

Here's a simple example. This:

```sql
SELECT PATINDEX('%[a-z][a-z][a-z]%', 'aa bb ccc dd eee ff')
```

returns:

```
    (No column name)
    ----------------
    7
```

...which is the index of the first occurrence of any 3-character word in the input string (keep in mind that SQL uses 1-based indexing... _*shudder*_).

Some obvious limitations of PATINDEX are:

*   It only supports simple patterns.
*   It can only match the first occurrence of a pattern.
*   It does not return the match it found.

We are going to work around these limitations and create a function that can take any input string and pattern, and replaces all matches of the pattern with something else.

## Fancy algorithm

So, how do we do this? One naive approach would be to just while-loop over the input and keep replacing until we don't find any more matches. Of course this doesn't work, because then replacing A with AX would send us into an infinite loop. We need to make sure we only hit each match once, and exactly once. There can be multiple matches in the input, but PATINDEX will only give us the first one. This suggests cutting up the input string as we go.

Here's one way of going about it:

![](/assets/img/blog/2020/05/sql-patterns.png)

This algorithm finds the first occurrence of the pattern, appends everything before that to the output, appends the replacement, and then continues on the remaining string.

It works, but there is one crucial missing detail: we don't know anything about the pattern that is being matched aside from its start index, because that's the only output PATINDEX gives us. We don't even know how long it is, which means we can't know how much of the input string to cut and replace.

Sadly, the only way around it is to make this an input parameter, meaning the consumer of our function will be responsible for providing the pattern as well as the length of the matches it will produce. "Fortunately", PATINDEX does not support variable-length patterns, so any matches it produces will always be the same length.

## Code it up

Here's my implementation of this algorithm:

```sql
CREATE FUNCTION dbo.fnReplaceAll
(
    @input          VARCHAR(MAX),
    @match_pattern  VARCHAR(MAX),
    @match_length   INT,
    @replace_value  VARCHAR(MAX)
)
RETURNS VARCHAR(MAX)
BEGIN

    DECLARE @output     VARCHAR(MAX) = '',
            @input_copy VARCHAR(MAX) = @input,
            @match_ix   INT;

    SET @match_ix = PATINDEX(@match_pattern, @input_copy);

    WHILE @match_ix > 0
    BEGIN

        SET @output = @output + SUBSTRING(@input_copy, 1, @match_ix - 1) + @replace_value;
        SET @input_copy = SUBSTRING(@input_copy, @match_ix + @match_length, LEN(@input_copy));

        SET @match_ix = PATINDEX(@match_pattern, @input_copy);
    END

    SET @output = @output + @input_copy;

    RETURN @output;

END
```

Simple enough, and it gets the job done. It's a pure T-SQL function, which means it doesn't require the CLR and'll run on Azure SQL.

A little test:

```sql
SELECT dbo.fnReplaceAll('a b c bb c a b aabbccbca a', '%b%', 1, 'bx')
```

...which correctly returns:

```
    (No column name)
    ----------------
    a bx c bxbx c a bx aabxbxccbxca a
```

It's not exactly regex, but hey, it still has its uses.
