---
title: Mutation testing and continuous integration
created_at: 2015-05-01 11:56:02 +0200
kind: article
publish: false
author: Andrzej Krzywda
newsletter: :arkency_form
---

Mutation testing is another form of checking the test coverage. As such it makes sense to put it as part of our Continuous Delivery process. In this blog post I’ll show you how we started using mutant together with TravisCI in the RailsEventStore project.

<!-- more -->

In the last blogpost I explained why I want to introduce mutant to the RailsEventStore project. It comes down to the fact, that RailsEventStory, despite being a simple tool, may become a very important part of a Rails application. 

RailsEventStore is meant to store events, publish them and help building the state from events (Event Sourcing). As such, it must be super-reliable. We can’t afford introducing breaking changes. Regressions are out of question here. It’s a difficult challenge and there’s no silver bullet to achieve this.

We’re experimenting with mutant to help us with ensuring the test coverage. There are other tools, like simplecov or rcov but they work on much worse level of precision. 

Another way of ensuring that we don’t do mistakes is to rely on automation. Whatever can be done automatically, should be done automatically. A continuous integration server is a part of the automation process. We use TravisCI here.

Previously, TravisCI just run `bundle exec rspec`. The goal was to extend it with running the coverage tool as well.

When I was experimenting with mutant and run it for the first time, I saw about 70% of coverage. That was far from perfect. However, it was a good beginning. My idea was to introduce mutant as part of the CI immediately - with the first goal being that we don’t get worse over time.

Mutant supports the `—score` option:

```
$ mutant -h
usage: mutant [options] MATCH_EXPRESSION …
Environment:
        —zombie                     Run mutant zombified
    -I, —include DIRECTORY          Add DIRECTORY to $LOAD_PATH
    -r, —require NAME               Require file with NAME
    -j, —jobs NUMBER                Number of kill jobs. Defaults to number of processors.

Options:
        —score COVERAGE             Fail unless COVERAGE is not reached exactly
        —use STRATEGY               Use STRATEGY for killing mutations
        —ignore-subject PATTERN     Ignore subjects that match PATTERN
        —code CODE                  Scope execution to subjects with CODE
        —fail-fast                  Fail fast
        —version                    Print mutants version
    -d, —debug                      Enable debugging output
    -h, —help                       Show this message
```

I didn’t read this output carefully, at first. I assumed that the score option is there to check if my coverage is equal or higher than the expected coverage. When I run the tests and checked the exit code (`echo $?`) afterwards I saw the result being 1 (a failure).

I assumed that something was broken and went to the mutant sources to find this [here](https://github.com/mbj/mutant/blob/7529b724c4409fdeb73c9a0fe6390ec7b5e4946c/lib/mutant/result.rb#L95):

```
#!ruby

      # Test if run is successful
      #
      # @return [Boolean]
      #
      # @api private
      #
      def success?
        coverage.eql?(env.config.expected_coverage)
      end
```

**Raising the coverage bar**

This must have been a mistake, I thought. Why would you ever want to assume that the coverage is equal to expected coverage. Being higher than the expected coverage is a good thing, right? Actually, it’s not a good thing. As I learnt from Markus (the author of mutant), this setting is intentional. The reason for that is that you want to fail in both cases - when the current coverage is lower than expected - that’s clear. You also want the build to fail, when it’s higher. Why? Because otherwise you may miss the point of time when you improved the coverage. Later on, you may have reduced again. You never noticed that the expected coverage should be raise. If I got it correctly, this technique is called “raising the bar”. After this explanation it made a perfect sense to me.