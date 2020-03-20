---
layout: post
title: Rapidly Distinguishing Between Variable-Layout Messages Using SIMD
---

Some years ago, at a now-defunct company, I wrote a nifty SIMD scanner to quickly discover "interesting" fields in a complex message format.
After getting the parse time around a microsecond, I considered my work finished and moved on.

I recently realized most of these messages probably came in some set of fixed layouts instead of being truly dynamic.
Instead of searching the whole message for individual entries,
it should be possible to check for a given layout by examining a small subset of bytes.

It turns out this can be done very, very, efficiently - costing about ~40 cycles per message.

The full code can be found [here](https://github.com/vgatherps/discriminant)

### Benchmarks preview
Since everybody loves benchmarks, here's a preview of the final results (in cycles):

|Test|Cached|Uncached|
|----|------|--------|
|Discriminant|42|355|
|Discriminant w/o length checks| 32 | 330|

### A basic message format
Let's consider a simplified version of this message format. Each field consists of:

1. A 1-byte field header with the value `'@'`
2. A 3-byte field identifying field, without the byte `'@'`, which is slightly expensive to validate
3. Arbitrary binary data NOT containing the byte `'@'`

As an example consider the message: `'@=a:abcd@=b:text@=c:xy'`, specifically the first field.
* It starts with the header `'@'`
* It has the 3-byte identifier  `'=a:'`
* It has the message body `'abcd'`

Importantly, there is no message length field in the header (so the message must be scanned for them) and the identifying field isn't free to parse.

### Checking fixed bytes to validate a message

Let's pretend we have three possible message layouts:
* `"@=a:abcd@=b:text@=c:xy"`
* `"@=a:abc@=b:text@=c:xy"`
* `"@=a$123@=b:text@=c:xy"`


We only have to check the 4th character for one of `'$', ':'` and the 9th for one of `'b', '='`. Since these uniquely determine the message, we will know the whole layout just from these checks

Naively, we could load the byte at the nth index of the message, check it against the discriminators, and branch into possible subpaths:

``` cpp
if (message[3] == '$') {
    return TYPE_3
} else {
    if (message[8] == '=') {
        return TYPE_1
    } else {
        return TYPE_2
    }
}
```

This is undesireable for two reasons:
* If branch prediction is hard, we will waste a lot of time in missed branches
* If branch prediction is easy, the predictor might "learn" that a very important message never comes
* This either requires hardcoding or some constexpr/metaprogramming magic

The next thing one could do is aggregate comparison results on each message type:

``` cpp
char m3 = message[3];
char m8 = message[8];

bool is_message1 = (m3 == ':') & (m8 == '=');
bool is_message2 = (m3 == ':') & (m8 == 'b');
bool is_message3 = (m3 == '$') & (m8 == 'b');

if (is_message1) {
    return TYPE_1
} else if (is_message_2) {
    // and so on
}
```

Aside from the fact that the compiler will likely turn this into branch code, there's an obvious inefficiency in that we do many comparisons against `m3` and `m8`. If we had more message types, this problem would only get worse.

Since we repeatedly do the same operation to m3 and m8, we can replace this with a SIMD implementation to do all the work in one pass.

### Testing message types with a single SIMD pass

Fortunately, using SIMD to solve this doesn't look much different at all - we do the same thing, just with all the variants for `m3` and `m8` at once.

``` cpp
// Create a 16-byte vector with each byte set to message[3/8]
__m128i m3 = _mm_set1_epi8(message[3]);
__m128i m8 = _mm_set1_epi8(message[8]);

//16 byte simd vector containing {':', ':', '$', 0...}/{'=', 'b', 'b', 0...};
__m128i m3_checks = computed_m3_vector;
__m128i m8_checks = computed_m8_vector;

// do a byte-wise comparison of the m3 vector and the bytes to check against
// Result is a byte of all-ones if it matches and zero if it doesn't
__m128i m3_match = _mm_cmpeq_epi8(m3_checks, m3);
__m128i m8_match = _mm_cmpeq_epi8(m8_checks, m8);

// and together the result to see which message matches both
__m128i matches_both = _mm_and_si128(m3_match, m8_match);

// For each byte that is nonzero (means it matches), set the corresponding bit to 1
int matches_bitmask = _mm_movemask_epi8(matches_both);

/// Mask off irrelevant upper bits
matches_bitmask &= 0b111;

// Handle matches layout here
```

Walking through a pass where we validate the message type 1, we have:

1. `m3 = _mm_set1_epi8(message[3]) = {':', ':', ':'} // repeat for m8`
2. `m3_checks = {':', ':', '$', 0....} // repeat for m8`
3. `m3_match = _mm_cmpeq_epi8(m3_checks, m3) = {-1, -1, 0...}`
4. `m8_match = _mm_cmpeq_epi8(m8_checks, m8) = {-1, 0, 0...}`
5. `matches_both = _mm_and_si128(m3_match, m8_match) = {-1, 0, 0...}`
6. `matches_bitmask = _mm_movemask_epi8(matches_both) = 1`

Leading to the only set bit being the first 1, the result we expected.

### Doing this in a generic way

The above is also trivially extendable to a generic input - all we have to do is put each discriminant vector into an array and loop over them:

``` cpp
// message is a char * to our message
// offsets_to_check is a list of the offsets in the message
// bytes_to_check is a list of vectors with the discriminating characters in them
__m128i matches_so_far = _mm_set1_epi8(1);
for (int i = 0; i < checks_to_do; i++) {
    int offset = offsets_to_check[i];
    __m128i message_byte = _mm_set1_epi32(message[offset]);
    __m128i bytes = bytes_to_check[i];
    __m128i match = _mm_cmpeq_epi8(bytes, message_byte);
    matches_so_far = _mm_and_si128(match, matches_so_far);
}
```

### Ignoring certain fields
For some messages, we have no opinion on bytes at a specific location. For example, let's modify our third message so that the 9th character is contained within a message:

* `"@=a:abd@=b:text@=c:xy"`
* `"@=a:ab@=b:text@=c:xy"`
* `"@=a$123456@=b:text"`

Since the sender can vary the 9th character now, we can't utilize it in a check. Therefore, we need some way to specify that we don't care about the result of a specific match.

This can be trivially solved with the addition of an ignorance vector - all matches are or'ed against this vector to set certain matches to unconditional truth.
If we want to ignore a value, the byte has all 1s and forces a match. Otherwise, it contains zeros and does nothing.

``` cpp

// bytes_to_ignore is a list of vector masks to ignore characters
__m128i matches_so_far = _mm_set1_epi8(1);
for (int i = 0; i < checks_to_do; i++) {
    int offset = offsets_to_check[i];
    __m128i message_byte = _mm_set1_epi32(message[offset]);
    __m128i match = _mm_cmpeq_epi8(bytes_to_check[i], message_byte);
    match = _mm_or_si128(match, bytes_to_ignore);
    matches_so_far = _mm_and_si128(match, matches_so_far);
}
```

### Handling message length
Right now, this check is only valid when each check lies within each message. Let's add a fourth, shorter message to check:

* `"@=a:abd@=b:text@=c:xy"`
* `"@=a:ab@=b:text@=c:xy"`
* `"@=a$123456@=b:text"`
* `"@=a#xcy"`

Right now, we would to unable to run our checks against this message since there's no 9th character to read. One could maintain an oversized buffer always refilled with garbage data, but for realistic uses that can get expensive (one has to ensure there's no stale data that could confuse a layout check).


We also run into the problem of branching again - filling the code with branches against message length is another avenue by which we can mistrain the branch predictor.
What immediately comes to mind is tracking the validity of each comparison with a running mask (like we track matches), and branchlessly redirecting oversized reads to garbage data.

With the help of `cmov` (conditional data movement), and some helpful assumptions:

* We never receive zero-length messages (or can reject them beforehand)
* All offsets are passed in increasing order
* If a certain offset can be ignored by a specific layout, that layout does not need to be that long

This can remain branch-free.

The first assumption means that we can always read from offset 0. If we see that an offset goes past the end of a message, we can set the offset to zero in a branchless fashion.

The second will allow for optimizations in how we track what messages are invalid or not.
Specifically, for each offset `O`, if it is ignored by a layout and all deeper offsets are ignored, that layout is allowed to contain less than `O` bytes.
For each offset, we can generate a bitmask with bits set to 1 if the message must contain `O` bytes, and 0 otherwise.

Since we receive the offsets in increasing order, this mask contains less and less 1s as the offsets increase.
If on any offset, we find the message to be too long, we or the current length mask into our running mask, and otherwise or 0 into it.

This approach doesn't abort early like a branching approach would, but also prevents us from getting stuck at the mercy of a mistrained branch predictor.

``` cpp
// length is the length of the message
__m128i matches_so_far = _mm_set1_epi8(1);
int mask_too_short = 0;
for (int i = 0; i < checks_to_do; i++) {
    int offset = offsets_to_check[i];
    bool too_short = offset >= length;

    // Assume these are branchless - in the actual code I use some asm to force this
    // Since GCC is "smart" enough to use branching code
    offset = too_short ? 0 : offset;
    mask_to_short |= (too_short ? is_this_long[i] : 0);

    __m128i message_byte = _mm_set1_epi32(message[offset]);
    __m128i match = _mm_cmpeq_epi8(bytes_to_check[i], message_byte);
    match = _mm_or_si128(match, bytes_to_ignore);
    matches_so_far = _mm_and_si128(match, matches_so_far);
}
```

### Moving some of the work to compile-time
One of the biggest performance boosts came from moving the functions which build my checking datastructures to compiletime.
While this deserves it's own post, I'll give a high level overview of how moving to compile-time makes things better.

The biggest change is in performance - by generating the checking datastructure at compile-time, the compiler has more optimization opportunities.
The biggest advantage in my test is that it's able to unroll the loop and constant-propagate much of the data into the code.
For example, the compiler can see that certain ignore masks are always zero and can eliminate the OR operation.

Another advantage is that I could make certain invalid checks compile-time failures. For example, if somebody wanted to ignore an entire check,
or had two redundant layouts to check, this would fail to compile.

Overall, my experience with constexpr was miles away from my past life in crazy template metaprogramming and greatly improved my opinion of modern C++.

### Benchmarks

To see the performance, I benchmark two cases against a 729-byte message:

1. The discriminant
2. The discriminant without length checks

in two cache scenarios:

1. My message is contained in the cache
2. I've totally evicted my message from the cache

These cases are both common - I might be processing my message after some code has been run on it, and would expect it to be in the cache. I might also be the first code to touch it from say a DMA buffer, where I would expect at best the find data in the L3 cache.

The goal here is to beat 1 microsecond spent in the scanner, ~3k cycles.
The results from the discriminant, in cycles, without accounting for measurement overhead, are:

|Test|Cached|Uncached|
|----|------|--------|
|With length checks|42|355|
|Without length checks|32|330|

The results are pretty expected - and the discriminant costs are what I would expect from the generated code. Further, although doing the branchless length checking doubles the cost, it's an exceptionally minor cost in the best case.

An interesting note is that the discriminant only pays the cost of 1 cache miss even though the entire message in in RAM.
Since the discriminant deterministically schedules a set of loads no matter what the data is, it can fetch all the results in parallel.
Effectively, it only sees the cost of one cache miss.

Overall, the discriminant handily trounces the scanning method (~3k cycles) in the average cost, and likely has much more consistent latency tails.
In a followup post, I'll dive into these latency measurements as well as measure the cost of branching vs branchless on variable messages.

### Is this actually a useful idea?
This is a pretty risky idea unless you can really trust the sender not to start changing layouts without advance notice. Otherwise, it's completely possible to have the layout change without affecting the discriminating bytes.
As an intellectual exercise, this was an interesting insight into how fast the CPU can be when you work with the machine instead of against it, as well as how powerful optimizing compilers are.

It's also debatable whether it's better than devising a set of checks with different offsets per message. For my problem, most message formats only different by a byte or two at most, so most static regions (headers and constant data) partially overlapped. In other cases, with totally different formats, this approach would end up ignoring most checks.
