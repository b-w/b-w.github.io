---
layout: post
title: "Some 70-461 exam tips"
categories: blog
---

I passed the [Microsoft 70-461 exam](https://www.microsoft.com/en-us/learning/exam-70-461.aspx) today with a score of 885 points. Obviously I can't speak about the contents of the exam (yay NDA), but here are some general tips on passing this exam:

*   The [training kit](https://www.microsoftpressstore.com/store/training-kit-exam-70-461-querying-microsoft-sql-server-9780735666047) is good, but not perfect. Some things are explained poorly, or simply omitted. You can use the training kit as your primary study material (I did), but don't be afraid to use additional resources such as MSDN to fill any gaps that it leaves.
*   Don't skip the questions and practice assignments at the end of each chapter! It's good to get some hands-on experience.
*   The practice exam on the training kit CD is completely and utterly useless. It simply does not even come _close_ to the real exam. Not in terms of the types of questions, and certainly not in terms of difficulty. So don't bother with that one. I don't have any experience with any third party practice exams (they cost money, yo) so I can't comment on those.
*   Practice. By which I mean: actually open up that SSMS and tinker with queries. You won't make it on theory alone. Nothing beats real-world experience, but for those things that you don't run into on a day-to-day basis (looking at you, XML), just practice with them in your own time. You won't get a good feeling for how this stuff works just by reading about it.
*   Know your syntax. The exam isn't just multiple-choice; you _will_ be writing T-SQL statements.
*   Know the pros and cons of things. Know when to use one option, and when to use another. For example, the SNAPSHOT isolation level does not require locking, but uses tempdb storage. Such a trade-off might be the subject of a question.
*   In similar vain: know the prerequisites and limitations of things. For example, to add a clustered index to a view, the view needs to have the WITH SCHEMABINDING option.
*   Carefully read each question, and identify the various requirements. It's easy to overlook words like "descending", "not", or "first", but trust me: they matter. After giving your answer, go back to the requirements and double-check to see if you've satisfied each one.
*   For multiple-choice questions: spot the difference! There will be questions that have you choosing between blocks of T-SQL code. Rather than trying to parse and understand each block separately, focus on where the various blocks differ from each other.
*   Also for multiple-choice: start by eliminating the obviously wrong options.
*   Take your time. It took me 1,5 hours to get through all questions, leaving me with 30 minutes to review stuff I had trouble with. If you know your stuff, the clock should be no issue for you.
*   The exam isn't easy, but it is fair. For the most part, it doesn't rely on hammering you with really obscure features and edge-cases. I only had 2-3 questions that made me go "WTF". Again, as long as you have a decent feel for all of the different topics, you shouldn't have too much trouble.

If you're currently studying for this exam: good luck!
