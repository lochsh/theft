# theft: property-based testing for C

theft is a C library for property-based testing. Rather than checking
the results with specific input, properties are asserted ("for any
possible input, [condition] should hold"), and theft searches for
counter-examples. If it finds a combination of arguments that causes the
property to fail, it will search for simpler versions of those arguments
that still fail, and then print the minimal failing input.

[Property-based testing][pbt] stresses programs differently than tests
biased by understanding how the program "should" work. Like using fuzz
testing to find crashes or security vulnerabilities, this can discover
edge cases that have not been covered by unit tests. It also generates
thousands of tests with just a few lines of code, so it's a great way to
get quick feedback on code that is rapidly evolving.

theft is distributed under the ISC license.

I have a [blog post][] about one of the theft tests used in
[heatshrink][], and its repo has [several other property-test examples][ex].

[pbt]: https://spin.atomicobject.com/2014/09/16/property-based-testing/
[blog post]: https://spin.atomicobject.com/2014/09/17/property-based-testing-c/
[heatshrink]: https://github.com/atomicobject/heatshrink
[ex]: https://github.com/atomicobject/heatshrink/blob/master/test_heatshrink_dynamic_theft.c

## Installation & Dependencies

theft does not depend on anything beyond C99. Its tests use
[greatest][], but there is not any coupling between them. It also
contains implementations of the [Mersenne Twister][mt] PRNG and the
[FNV-1a][fnv] hashing algorithm - see their files for copyright info.

[greatest]: https://github.com/silentbicycle/greatest
[mt]: http://www.math.sci.hiroshima-u.ac.jp/~m-mat/MT/emt.html
[fnv]: http://www.isthe.com/chongo/tech/comp/fnv/


To build:

    $ make

To build and run the tests:

    $ make test

This will produce example output from some trivially failing properties,
and confirm that failures have been found.

To install libtheft and its headers:

    $ make install    # using sudo, if necessary


## Example Properties

### In a data compression library:

+ For any input, compressing and uncompressing it should produce output
  that matches the original input.

+ For any input, the compression output should never be larger than the
  original input, beyond some small algorithm-specific overhead.

+ For any input, the uncompression state machine should never get stuck;
  it should always be able to reach a valid end-of-stream state once
  the end of input is reached.

### In a parser:

+ For any input, it should output either a successful parse with a valid
  parse tree, or error information.

+ For any valid input (generated by randomly walking the grammar), it
  should output a valid parse tree.

### In a flash memory wear-leveling system:

+ For any sequence of writes (of arbitrary, bounded size), no flash page
  should have significantly more writes than the others.

### In a datagram-based network:

+ For any order of receiving packets (including retransmissions), all
  packets should eventually be received and acknowledged, and every
  packet should be checksummed once, in order.
  
### In data structure implementations:

+ For any sequence of insertions and deletions, a balanced binary tree
  should always stay approximately balanced.
  
+ For any input, a sorting algorithm should produce sorted output.


## Usage

First, define a property function:
```c
    static theft_trial_res
    prop_encoded_and_decoded_data_should_match(buffer *input) {
        // [compress & uncompress input, compare output & original input]
        // return THEFT_TRIAL_PASS, FAIL, SKIP, or ERROR
    }
```

Then, define how to generate the input argument type(s) by providing a
struct with function pointers. (This definition can be shared between
all properties that have the same input type.) For example:

```c
    static struct theft_type_info random_buffer_info = {
        .alloc = random_buffer_alloc_cb,    // allocate random instance
        .free = random_buffer_free_cb,      // free instance
        .hash = random_buffer_hash_cb,      // get hash of instance
        .shrink = random_buffer_shrink_cb,  // simplify instance
        .print = random_buffer_print_cb,    // print instance
    };
```

All of these callbacks except 'alloc' are optional. For more details,
see "Callbacks" below.

Finally, instantiate a theft test runner and pass it the property and
type information:

```c
    struct theft *t = theft_init(0);   // 0 -> auto-size bloom filter

    // Configuration for the property test
    struct theft_cfg cfg = {
        // name of the property, used for failure messages (optional)
        .name = __func__,

        // the property function under test
        .fun = prop_encoded_and_decoded_data_should_match,

        // list of structs with argument info; the property function
        // will be called with this many arguments
        .type_info = { &random_buffer_info },

        // number of trials to run; defaults to 100
        .trials = 1000,
    };

    // Run the property test. Any failures will be printed, with seeds.
    theft_run_res result = theft_run(t, &cfg);
    theft_free(t);
    return result == THEFT_RUN_PASS;
```

The result will indicate whether it was able to find any failures. An
optional progress callback and report struct can be used to get more
detailed results. (See the optional fields for `theft_cfg` in
`theft_types.h`.)


## Callbacks

All of the callbacks are passed a `void *env` argument from the
`theft_run` function, which can be used to pass an environment with
arbitrary state along. For example, it can pass a custom memory
allocator to alloc, or limit information used while generating the
instance. (This combination of a callback & environment pointer is also
known as a closure.) If its contents vary from trial to trial and
influence the property test, it should be considered another input and
hashed accordingly.

### alloc - allocate an instance from a random number stream

```c
    void *(theft_alloc_cb)(struct theft *t, theft_seed seed, void *env)
```

Use a stream of random numbers to construct an instance of the argument.
More random numbers can be requested with `theft_random()`. This is
called with a known seed, so the same instance can be constructed again
if necessary. (These streams of random numbers may not be consistent
across versions of the library, though.)
   
This is the only required callback.

### free - free an instance and any associated resources

```c
    void (theft_free_cb)(void *instance, void *env);
```

Free the memory and other resources associated with the instance. If not
provided, theft will just leak resources.

### hash - get a hash for an instance

```c
    theft_hash (theft_hash_cb)(void *instance, void *env);
```

Using the included `theft_hash` functionality, get a hash value for a
given instance. This will usually consist of `theft_hash_init`, then
calling `theft_hash_sink` on the instance's contents, then returning the
result from `theft_hash_done`.
   
If provided, theft will use these hashes to avoid re-testing
combinations of arguments that it has already tried. Note that if the
contents of `env` impacts how instances are constructed / simplified, it
should also be fed into the hash.
   
### shrink - produce a simpler copy of an instance

```c
    void *(theft_shrink_cb)(void *instance, uint32_t tactic, void *env);
```

For a given instance, return either a pointer to a simplified copy of
the instance, or special `THEFT_DEAD_END` or `THEFT_NO_MORE_TACTICS`
values. This must eventually bottom out, e.g., simplifying a list by
dropping the first value will eventually produce an empty list and
return `THEFT_DEAD_END`. shrink must allocate and return a fresh copy,
rather than modifying the instance passed in.
   
The 'tactic' argument selects which approach is tried. It starts at 0
and increases until a tactic successfully shrinks or
`THEFT_NO_MORE_TACTICS` is returned. For more info, see *Shrinking*
below.
   
If not provided, theft will just report the initially generated
counter-example arguments as-is. This is equivalent to a shrink callback
that always returns `THEFT_NO_MORE_TACTICS`.

### print - print a string based on a random instance

```c
    void (theft_print_cb)(FILE *f, void *instance, void *env);
```

Print the instance to a given file stream, behaving like: 

```c
    fprintf(f, "%s", instance_to_string(instance));
```

If not provided, theft will just print the random number seeds that led
to discovering counter-examples.


## Shrinking

Once theft has found input that causes the property to fail, it will try
to 'shrink' it to a minimal example. It can be hard to tell what aspect
of the original random arguments caused the property to fail, but
shrinking will eliminate irrelevant details, leaving input that should
point directly at the problem. (These simplified arguments may also be
good test data for unit/regression tests.)

The shrink callback is given a tactic argument, which chooses between
ways to simplify the instance: "Try to simplify this using tactic #2".
These should be ordered by how much they simplify the instance, because
shrinking by bigger steps helps theft to converge faster on minimal
counter-examples.

For a list of numbers, shrinking tactics could include:

+ Discarding the first half of the list
+ Discarding the second half of the list
+ Dividing all the numbers by 2 (with integer truncation)
+ Discarding the first value
+ Discarding the last value
+ Discarding the middle value

The first 3 shrink the list by much larger steps than the others, which
will only be tried once first 3 discard whatever details are causing the
property to fail. Then, if a later tactic leads to a simpler failing
instance, then it will try the earlier tactics again in the next pass --
they may no longer lead to dead ends.

Shrinking works by breadth-first search over all arguments and all of
their shrinking tactics, so it will attempt to simplify all arguments
that have shrinking behavior specified. While this tends to find local
minima rather than the absolute simplest counter-examples, it will
always report the simplest counter-examples it finds. If hashing
callbacks are provided, it will avoid revisiting parts of the state
space that it has already tested.

The shrink callback can also vary its tactics as the instance changes.
For example, exploring changes to every individual byte in a 64 KB byte
buffer is probably too expensive, but could be worth trying once other
tactics have reduced the buffer to under 1 KB. Similarly, the contents
of `env` can influence shrinking -- if the property function stores
coverage info from the most recent trial in `env`, it can focus
shrinking on parts of the input that are actually being used. (In that
case, be sure to include the env data in the hash.) Since the shrink
callback's tactic argument is just an integer, its interpretation is
flexible.

The requirement for shrinking is that the same combination of `env`
info, argument(s), and shrinking tactic number should always simplify to
the same instance, and therefore lead to the property function having
the same result.


# Running and Reporting

`theft_run` has an optional 'progress_cb' argument, which takes a
pointer to a function to call with details after every trial. The
callback should return `THEFT_PROGRESS_CONTINUE`, or
`THEFT_PROGRESS_HALT` to halt a test run early. This can be used to stop
searching if there are too many duplicates, to print '.' characters to
show progress every N iterations of a slow test, to halt after a certain
number of failures have been found, etc. If not set, theft will default
to a progress callback that just returns `CONTINUE`.

`theft_run` also has an optional argument, 'report', which takes a
pointer to a `theft_trial_report` struct. If non-NULL, this will be
populated with more detailed info about the test run's result, which
only indicates whether there were any failures. Currently, it includes
counts for passes, failures, trials skipped at user request, and trials
skipped because the arguments are probable duplicates.
