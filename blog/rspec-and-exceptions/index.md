---
title: RSpec and exceptions
date: 2017-09-12
tags:
  - tech
  - ruby
  - testing
  - today-i-learned
lang: en
---

Today I learnt that these two tests are, not only non-functionally different, but functionally different too:

```ruby
# Test A:

it "should run without raising an exception" do
  some_code_to_test
end

# Test B:

it "should run without raising an exception" do
  expect {
    some_code_to_test
  }.not_to raise_error
end
```

Like JUnit, rspec reacts to tests which raise exceptions by failing the test, right? So these two tests should (functionally) behave identically?

Wrong!

Well, _sometimes_ wrong.

The difference (or rather, at least one of the differences — perhaps there are more) lies in `SystemExit`. If `some_code_to_test` raises this, typically by calling `Kernel#exit`, then the test runner stops, treats this test as successful, doesn’t show the output of this test, and silently skips all subsequent tests:

```text
it "runs test 1" do
end

it "should run without raising an exception (2)" do
  some_code_to_test
end

it "runs test 3" do
end

// and then run it:

rachel@shinypig test$ bundle exec rspec --format doc spec/my_spec.rb

Kernel
  runs test 1

Finished in 0.00089 seconds (files took 0.08703 seconds to load)
2 examples, 0 failures
```

which reports that it ran 2 examples, but only shows one of them, and doesn’t even mention the one that it skipped completely.

On the other hand, if we wrap the code being tested using “expect … not_to raise_error”:

```text
rachel@shinypig test$ bundle exec rspec --format doc spec/my_spec.rb

Kernel
  runs test 1
  should run without raising an exception (2) (FAILED - 1)
  runs test 3

Failures:

1) Kernel should run without raising an exception (2)
     Failure/Error: expect { some_code_to_test }.not_to raise_error

expected no Exception, got #<SystemExit: exit> with backtrace:
         # ./spec/my_spec.rb:4:in `exit'
         # ./spec/my_spec.rb:4:in `some_code_to_test'
         # ./spec/my_spec.rb:11:in `block (3 levels) in <top (required)>'
         # ./spec/my_spec.rb:11:in `block (2 levels) in <top (required)>'
     # ./spec/my_spec.rb:11:in `block (2 levels) in <top (required)>'

Finished in 0.01099 seconds (files took 0.07536 seconds to load)
3 examples, 1 failure

Failed examples:

rspec ./spec/my_spec.rb:10 # Kernel should run without raising an exception (2)
```

So now it runs all three tests, showing all three results, including one failure.

To be honest I’m surprised that the rspec test runner doesn’t deal with this natively: if a test raises an error, it should always be a failure (to my mind anyway), including `SystemExit`. Maybe there’s a bug / pull request for rspec somewhere discussing this, where the idea was rejected. I might go digging…
