---
layout: post
title:  "One Assertion per Unit Test...or Not?"
date:   2019-02-20 23:04:10 +0100
---
**TL;DR:** In unit testing, there seem to be many good reasons to bundle multiple assertions in 1 test. More often than not, though, I found that these reasons prevented me from reaching my goal: fast and focused micro tests that tell me exactly what’s wrong in the case of failure. As a consequence, I almost always stick to 1 assertion per unit test.

## So, this topic again

It came up several times for me in the course of the last months. First, there was [a tweet by Uncle Bob](https://twitter.com/unclebobmartin/status/1078695335935979520) that spawned a short-lived but lively discussion in which many interesting statements popped up. Only a few days later, I was asked to review some code changes that came with a few tests. Some of these tests contained several assertions that were related conceptually; I would have preferred them to be in separate tests still. Again, there was some discussion during which I failed to convince the author to split up the tests. Since that discussion only touched some points of my “unit testing philosophy”, I thought it might be worthwhile to sum up my point of view here.

When it comes to unit testing, I like to follow the [FIRST principles](https://agileinaflash.blogspot.com/2009/02/first.html) and write fast, focused micro tests. For consistency and readability, I tend to use the [Arrange-Act-Assert pattern (3A)](https://xp123.com/articles/3a-arrange-act-assert/) to structure these tests. The tests should, in the case of failure, point exactly to WHAT is wrong (i.e., the difference between the expected and the actual state); ideally, they should even hint at the WHY (i.e., the technical reason)—firing up the debugger and poking at the problem takes a lot of time that I’d rather spend on writing code. In other words, my tests should have exactly 1 reason to fail (granted, this can be hard to achieve some times). And that reason should be easy to capture in 1 assertion, right?

## Why Have More Than One Assertion in the First Place?

There are various reasons why tests end up having several assertions, some more convincing than others. They were often mentioned to me during conversations and for the longest time, some of them guided my writing tests; over time, however, I have realised that they result in more problems then benefits.

Reasons for using more than 1 assertion are not mutually exclusive (e.g., I can try to save time by performing multiple transitions); they do imply slightly different approaches, though, each with its own implications. So let’s look at them separately.

# 1) The Test Checks Several Values Describing a Complex State

This is actually a legit reason. All the values might have a relationship that forms one complex picture; they may even be already grouped in a struct or a class for that very reason. So if one or more of the values are slightly off, there must be something wrong in that relationship and I want to verify them all in one go.

Doing so with multiple assertions has a clear downside, though. Most unit testing frameworks will stop evaluating a test after the first failing assertion. But what about the rest? I might not be getting the full picture regarding WHAT is wrong should one of these tests fail. As a result—and that's maybe the worst part—I might need to fix the production code in several turns. The worst case scenario can be described as follows:

> For n assertions in the test, I need to run the test n+1 times before it’s green again.

In the 1st run, the 1st assertion fails; I fix it… in the nth run, the nth assertion fails. I fix it. The next run is green again.

What’s the solution then? In my experience, a relationship like the one described here lends itself well to being verified in a custom assertion. Such assertions can produce a nicely formatted diff of actual and expected values. This generally beats the noisy output of multiple failing assertions (in case my unit testing framework does keep going after the first one).

# 2) The Test is Trying to Save Time by Checking Various Things

Sometimes, the unit under test (UUT) needs a lot of setup, including numerous dependencies (collaborators, services, etc.) to be testable. As a result, a test is costly to run. In such situations, I often found myself tempted to cram various things into the same test— _get stuff done once the unwieldy beast is finally up and running._

But hang on, isn't the test trying to tell me something here? If my UUT takes so long to get into a usable state, won't this affect my application at runtime? Maybe not, maybe we're talking about a unit that is only created once. But hey, can I say with certainty that this will always be the case? Better get back to the drawing board.

UUTs rarely need a lot of time to be initialized all on their own. Some services may need to be fired up in the background, some collaborators need to instantiated and prepared, files need to be read (RED FLAG!). My unit test doesn't care about any of this, however; it just wants to verify that, given a defined situation, the UUT performs one action properly with the expected outcome. I don't care about these services, collaborators, etc. either in the context of the test, by the way; it can be a huge pain and a waste of time having to look at all of them with a looking glass (a.k.a. the debugger) to find out the failure reason.

With this in mind, I can start digging. Is this service really needed here? What about that collaborator? If it is needed, what's the expected behavior? A fake (or stub) implementation mimicking that behavior should be faster to set up and would be more reliable—if my dependencies have their own dependencies, I have even more dependencies to rely on. Can the aspect of the UUT my test is focused on be extracted into another unit to minimize the dependencies? Guess I’m designing now…

All this aside, there’s something else worth considering: in the time it takes to fire up the debugger to find out WHAT is wrong, I (or my CI system) can run plenty of more focused and thus more informative tests; even if they take a while to run.

# 3) The Test Performs and Verifies More Than One State Transition

Yes yes, I have done this, too. _While I'm in the area (i.e., the new state after the first transition), I might as well check this other action._ Sounds reasonable, right? Well, this kind of thinking has proven to be problematic more than once for me.

Here is a simple example:
```python
def test_multiple_transitions():
    u = UUT()               # ARRANGE 1
    u.doA()                 # ACT 1
    assert u.state() == 'A' # ASSERT 1

    u.doB()                 # ACT 2
    assert u.state() == 'B' # ASSERT 2

    ...
```

It’s pretty obvious that this test is not structured with the 3A pattern—there are at least 5. If ASSERT 1 fails, I can focus on the first 3 steps: ACT 1 has not transitioned the UUT to the expected state, so I have to check out that function. Hmmm, wait, what about ASSERT 2? Should my unit testing framework have stopped at the first failure, I cannot be sure whether ACT 2 would have worked. But could I even trust ASSERT 2 if it passes after a failure in the first transition? Either way, I need to make ASSERT 1 pass before I know what’s going on with the rest of the test.

Now, what about a failure in ASSERT 2? Since ASSERT 1 passed, I can be sure that ACT 1 has produced the right state. So the failure must stem from ACT 2, right? Right? No, not necessarily! ACT 1 might have had a side-effect on a collaborator or service that only affects the state in the context of ACT 2--- I know this sounds like a manufactured scenario, but almost every developer I know can tell stories that have a similar ring to them. So, no clear pointer to WHAT went wrong. Let me start my debugger, just to be sure… _sigh_.

If I write a test like this, isn't it likely I will write more? For the sake of the argument let’s say that from state `A`, I can perform both `doB()` and `doC()`. It is easy to write the test verifying the outcome of `doC()` by copying the structure of that first test. In fact it’s so easy, I found it becomes a habit way too quickly. Suddenly, I have multiple tests that will fail should something break w.r.t. `doA()`. Maybe this gives me more pointers to the failure reason in that case; but all of these tests only do part of their work and I don’t get any information about the other transitions.

All of this feels a bit like an anti-pattern, so let's rephrase it in a more general manor:
> Performing multiple transitions in a test means using 1 transition (i.e., `doA()`) as part of the ARRANGE step preparing for another (i.e., `doB()`). If the UUT has no other means to get to the state needed for the second transition (i.e., `A`), there is a hard dependency between both transitions. In this case, the dependee cannot be verified while the depender is broken. Should there be many of these dependencies, 1 broken transition can greatly reduce the expressiveness of the entire test suite.

Oh, and did I mention that such tests are more brittle? The example test performs (at least) 2 transitions. This fact also means that code changes in the respective area of the production code are twice as likely to affect this test. But let's talk about test maintenance separately…like now.

# 4) The Test Combines Checks to Reduce the Amount of Test Code That Needs to be Maintained

More tests mean more code to maintain, it’s hard to argue with that. Changes to my UUT’s setup or interface will most likely affect more tests if there are more of them. Especially if the tests are very similar in terms of what they do. It might seem like a good idea to merge some tests for that reason (if it weren't for all those pesky problems I mentioned above).

But remember [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)? I try to apply this principle to both production- and test code to keep maintenance cost to a minimum (as most developers probably do, too). Do many tests share the same setup code? Extract a helper function. Can similar code to calculate the expected state/value/outcome be found in multiple tests? A custom assertion should help to reduce repetition.

One concept I have come to appreciate in such situations is parameterized testing. The idea behind it is to re-use the test logic (i.e., the body of the test) while defining the different test cases as sets of input- and output data---there are libraries and tools in [Python](https://routley.io/tech/2017/08/09/parameterise-python-tests.html) , [Java](https://github.com/junit-team/junit4/wiki/Parameterized-tests) and many more languages. It gets a lot easier to add new cases while changes to the UUT lead to fewer changes in the tests.

Ultimately, my tests should help me maintain my production code. That of course means they shouldn't be hard to maintain themselves or make changes to the UUT harder. Other developers will be annoyed by my tests if that's not the case and might refrain from running them.

## Conclusion

The benefits of having lots of focused micro tests only became apparent to me when I wrote tests of a different kind—the kind that has several assertions, for example. We as developers seem to generally be in agreement that we want to strive for readable code that clearly reveals its intention. It seems only logical that the same kind of diligence should be applied when writing tests. But what about the output of a failing test? Well-named tests that fail can very precisely point me to the part of the code that I need to focus on; but only if it’s clear WHAT went wrong. At the same time, the surrounding tests that pass narrow down my focus even further by pointing out parts of the production code that behaves as expected. It’s hard to estimate the time and frustration of having to navigate between methods (even classes), of firing up the debugger and trying to understand what’s going on, these tests have saved for me. I’m very happy working in their presence and get a little uneasy in their absence.

Btw: I’m not trying to vilify debuggers here. They are brilliant tools in sooo many situations. But starting a debugger to gather the information a failing test could have provided is a waste of its abilities and my time.

Oh boy, look at that! I ended up writing much more than I set out to. But did I miss anything? Is there a good reason to write tests with multiple assertions that I did not mention? Are there more reasons to stick to 1 assertion? Let me know on [Twitter](https://www.twitter.com/derteta).
