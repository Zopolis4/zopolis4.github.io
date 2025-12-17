## Context

[Back in late 2023](https://github.com/Zopolis4/chromebrew/commit/2a7027a365a4b2c000da116e7482ac56ed139a12), I was working on a script to check the upstream versions of Chromebrew packages. This was going to replace our existing solution, which required individual entries to be manually added for each package.

That script, like many of my other projects at the time, ended up stalling at around 90% completion and probably would have remained that way for some time had it not been picked up by [uberhacker](https://github.com/uberhacker), who took my script and filled in the gaps, with the result being merged upstream as `tools/version.rb` in [#9501](https://github.com/chromebrew/chromebrew/pull/9501).

One of the reasons my attempt sat in a state of near-completion for as long as it did was due to the choice of a version comparison algorithm. With over 2000 packages, a simple `new_version > old_version` isn't gonna cut it. (Granted, the initial versions of my script used `new_version != old_version`, which isn't exactly perfect either)

When it comes to version comparison algorithms, the ingeniously-named `libversion` is the best around. Battle-tested as the core of [Repology](https://repology.org), it can handle just about any edge case you can think of, as well as several that you can't.

With that in mind, the first change I made to the upstream version of `tools/version.rb` was to swap the algorithm from `Gem::Version` to `libversion` in [#9913](https://github.com/chromebrew/chromebrew/pull/9913). This was achieved using bindings I had conveniently [prepared earlier](https://github.com/Zopolis4/ruby-libversion/commit/0936807a11088b160cae61f1ac09a6b91c0ffe64), as `libversion` is written in C, while Chromebrew is written predominantly in Ruby.

These bindings were relatively simple to implement, as Ruby has good support for calling into C code, and `libversion` has a simple interface. To prove it, here's the actual binding code:

```c
#include "ruby.h"
#include "libversion/version.h"

static VALUE rb_version_compare2(VALUE self, VALUE v1, VALUE v2) {
  return INT2NUM(version_compare2(StringValueCStr(v1), StringValueCStr(v2)));
}

static VALUE rb_version_compare4(VALUE self, VALUE v1, VALUE v2, VALUE v1_flags, VALUE v2_flags) {
  return INT2NUM(version_compare4(StringValueCStr(v1), StringValueCStr(v2), NUM2INT(v1_flags), NUM2INT(v2_flags)));
}

void Init_ruby_libversion(void) {
  VALUE Libversion = rb_define_module("Libversion");
  rb_define_singleton_method(Libversion, "version_compare2", rb_version_compare2, 2);
  rb_define_singleton_method(Libversion, "version_compare4", rb_version_compare4, 4);
  rb_define_const(Libversion, "VERSIONFLAG_P_IS_PATCH", INT2NUM(VERSIONFLAG_P_IS_PATCH));
  rb_define_const(Libversion, "VERSIONFLAG_ANY_IS_PATCH", INT2NUM(VERSIONFLAG_ANY_IS_PATCH));
  rb_define_const(Libversion, "VERSIONFLAG_LOWER_BOUND", INT2NUM(VERSIONFLAG_LOWER_BOUND));
  rb_define_const(Libversion, "VERSIONFLAG_UPPER_BOUND", INT2NUM(VERSIONFLAG_UPPER_BOUND));
}
```

The main issue we encountered with using `libversion` was actually just getting the C library to build and link against. It isn't shipped by most package managers, and while that was obviously trivial to resolve on our end by simply [adding it to Chromebrew](https://github.com/chromebrew/chromebrew/pull/9983), our CI runs on Ubuntu.

After looking into the process of adding a package to Debian, it became clear somewhere around page 45 of the 89-page Debian Packaging Tutorial that it would be [much easier](https://github.com/chromebrew/chromebrew/pull/11623) to just compile `libversion` from source.

While this worked, it was obviously not ideal, and when `deb_version` was proposed in [#13022](https://github.com/chromebrew/chromebrew/pull/13022) as a replacement due to its lack of a C dependency, I knew it was time to properly address the situation.

To that end, I decided to rewrite `libversion` in Ruby, so that my `ruby-libversion` gem would have something to fall back on if the C library wasn't available.

## ruby_libversion.rb

Excluding bindings, `libversion` is available in a few languages already:

* [repology/libversion](https://github.com/repology/libversion) (C)
* [repology/libversion-rs](https://github.com/repology/libversion-rs) (Rust)
* [lizmat/Version-Repology](https://github.com/lizmat/Version-Repology) (Raku)

There is also a comprehensive description of the algorithm `libversion` implements in the aptly-named `doc/ALGORITHM.md` file found in the `repology/libversion` repository. The actual algorithm is explained in about 30 lines, and is fairly intuitive.

Initially, I planned to translate the existing C implementation into Ruby on a line-by-line basis, and then go back and touch some areas up to take advantage of Ruby's capabilities.

This soon fell apart when I actually looked at the C implementation, as it comprises a little over 300 lines of largely undocumented code split across 8 files. The Rust version isn't much better, taking up a little over 500 lines of code across 5 files. The Raku version is down to about 230 lines in one file, which is a little better, but Raku doesn't exactly translate nicely into Ruby.

## Comparisons

I couldn't think of a good segue into this, but I'm going to go over the algorithm, explain it, and then compare the documentation, C implementation, and my Ruby implementation. I'm not going to look at the Rust and Raku implementations, because they weren't the ones I was rewriting from. Also, the Rust implementation is basically just a direct translation of the C implementation, unsafe behavior and all, while the Raku implementation isn't really comparable to anything else.

A quick aside: I'm comparing the implementation of `version_compare2`, so nothing about patchlevel or bound flags. I've edited all code and documentation extracts to reflect that, as well as adjusted some indentation in places, just to help things flow better. Everything I've said and will say applies to `version_compare4` as well, it would just make for a slightly messier read if I went into detail about it.

### Step 1

```
Version is split into separate all-alphabetic or all-numeric components.
All other characters are treated as separators.
Empty components are not generated.
```

In Ruby, this is pretty simple, using a regex to split the string into separate all-alphabetic or all-numeric components, and then going over that array and replacing any groups of non-alphanumeric characters with an empty string. That looks a little something like this, which is slightly more verbose than typical regex because I'm using Ruby's [Unicode Property](https://docs.ruby-lang.org/en/master/Regexp.html#class-Regexp-label-Unicode+Properties) syntax:

```ruby
# Separate the version string into all-alphabetic and all-numeric components, with all other characters treated as separators.
# Separators are recorded as empty strings, with consecutive separators being squeezed into a single empty string.
version_raw_components = version_string.scan(/[[:alpha:]]+|[[:digit:]]+|[^[:alnum:]]+/).map { _1.match?(/[^[:alnum:]]/) ? '' : _1 }
```

In the C implementation, the version isn't actually split, and something closer to a custom iterator is implemented, using the function `get_next_version_component`. The relevant parts of that function are as follows:

```c
size_t get_next_version_component(const char** str, component_t* component, int flags) {
	*str = skip_separator(*str);

	if (**str == '\0') {
		make_default_component(component, flags);
		return 1;
	}

	parse_token_to_component(str, component, flags);
```

Skipping non-alphanumeric characters is handled by the aptly-named `skip_separator` function, defined as such:

```c
static inline const char* skip_separator(const char* str) {
	const char* cur = str;
	while (my_isseparator(*cur))
		++cur;
	return cur;
}
```

The actual logic exists in `my_isseparator`, which checks that the character isn't alphanumeric and isn't a null terminator:

```c
static inline int my_isalpha(char c) {
	return (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z');
}

static inline int my_isnumber(char c) {
	return c >= '0' && c <= '9';
}

static inline int my_isseparator(char c) {
	return !my_isnumber(c) && !my_isalpha(c) && c != '\0';
}
```

The use of `make_default_component` is in padding, which is left as an exercise for the reader.

The separation into all-numeric and all-alphabetic components is handled by `parse_token_to_component`, which iterates over a string until it finds a character of a different type to the start:

```c
static void parse_token_to_component(const char** str, component_t* component, int flags) {
	if (my_isalpha(**str)) {
		component->start = *str;
		component->end = *str = skip_alpha(*str);
		
		// removed for brevity
	} else {
		component->start = *str = skip_zeroes(*str);
		component->end = *str = skip_number(*str);

		// removed for brevity
	}
}
```

```c
static inline const char* skip_alpha(const char* str) {
	const char* cur = str;
	while (my_isalpha(*cur))
		++cur;
	return cur;
}

static inline const char* skip_number(const char* str) {
	const char* cur = str;
	while (my_isnumber(*cur))
		++cur;
	return cur;
}

static inline const char* skip_zeroes(const char* str) {
	const char* cur = str;
	while (*cur == '0')
		++cur;
	return cur;
}
```

That brings us to 3 lines to describe it, 1 line to implement it in Ruby, and 49 lines in the C implementation.

Ruby's brevity is largely thanks to the [`String.split`](https://docs.ruby-lang.org/en/master/String.html#method-i-split) method, which is tailor-made for this, although being able to method chain [`Enumerable.map`](https://docs.ruby-lang.org/en/master/Enumerable.html#method-i-map) using [`String.match?`](https://docs.ruby-lang.org/en/master/String.html#method-i-match-3F) comes in handy as well. A strong standard library is what wins Ruby the day here.

### Step 2

```
Components are assigned ranks by their value:
  - PRE_RELEASE: known pre-release keyword (alpha, beta, pre, rc).
  - ZERO: numeric component equal to zero.
  - POST_RELEASE: known post-release keyword (patch, post, pl).
  - NONZERO: numeric component not equal to zero.
  - LETTER_SUFFIX: If the previous component is numeric, the current component is alphabetic, and the next component is not numeric.
```

In Ruby, this isn't too bad, with the only notable aspect being the minor hassle I have to go through to convert integers. Ruby's [`String.to_i`](https://docs.ruby-lang.org/en/master/String.html#method-i-to_i) method returns `0` if the input string is not an integer, which is not ideal here, so I have go to the long way around with [`Kernel.Integer`](https://docs.ruby-lang.org/en/master/Kernel.html#method-i-Integer), tell it to return nil instead of throwing an exception on invalid input by passing `exception: false`, then handle that with safe navigation before we can finally get to our `.zero?` call. The handling of `LETTER_SUFFIX` is also longer than I'd like, and is the only reason I have to pass in the previous, current, and next component when determining the rank:

```ruby
# If the current component starts with a pre-release keyword (case insensitive), it is the pre_release rank.
return :pre_release if components[1].downcase.start_with?('alpha', 'beta', 'rc', 'pre')

# If the current component is an integer and has a value of zero, it is the zero rank.
return :zero if Integer(components[1], exception: false)&.zero?

# If the current component starts with a post-release keyword, it is the post_release rank.
return :post_release if components[1].start_with?('post', 'patch', 'pl', 'errata')

# If the current component is an integer and has a nonzero value, it is the nonzero rank.
return :nonzero if Integer(components[1], exception: false)&.nonzero?

# If the previous component is an integer, the current component is alphabetic, and the next component is not an integer, the current component is the letter_suffix rank.
return :letter_suffix unless Integer(components[0], exception: false).nil? || components[1].match?(/[^[:alpha:]]/) || !Integer(components[2], exception: false).nil?
```

You might also notice the prescence of `errata` as a `POST_RELEASE` keyword, which is missing in the algorithm documentation. It's in all the implementations and tests though, so I've got [a PR](https://github.com/repology/libversion/pull/41) up to fix that.

Moving on to the C implementation, assigning the ranks is done in tandem with parsing the version string into components, so they have the convenience of already knowing if the component they're dealing with is alphabetic or numeric by the time they determine the rank, which does help:
```c
// the alphabetic bit
static int classify_keyword(const char* start, const char* end, int flags) {
	if (end - start == 5 && my_memcasecmp(start, "alpha", 5) == 0)
		return KEYWORD_PRE_RELEASE;
	else if (end - start == 4 && my_memcasecmp(start, "beta", 4) == 0)
		return KEYWORD_PRE_RELEASE;
	else if (end - start == 2 && my_memcasecmp(start, "rc", 2) == 0)
		return KEYWORD_PRE_RELEASE;
	else if (end - start >= 3 && my_memcasecmp(start, "pre", 3) == 0)
		return KEYWORD_PRE_RELEASE;
	else if (end - start >= 4 && my_memcasecmp(start, "post", 4) == 0)
		return KEYWORD_POST_RELEASE;
	else if (end - start >= 5 && my_memcasecmp(start, "patch", 5) == 0)
		return KEYWORD_POST_RELEASE;
	else if (end - start == 2 && my_memcasecmp(start, "pl", 2) == 0)  /* patchlevel */
		return KEYWORD_POST_RELEASE;
	else if (end - start == 6 && my_memcasecmp(start, "errata", 6) == 0)
		return KEYWORD_POST_RELEASE;

	return KEYWORD_UNKNOWN;
}

/* Special case for letter suffix:
 * - We taste whether the next component is alpha not followed by a number,
 *   e.g 1a, 1a.1, but not 1a1
 * - We check whether it's known keyword (in which case it's treated normally)
 * - Otherwise, it's treated as letter suffix
 */
if (my_isalpha(**str)) {
	++component;

	component->start = *str;
	component->end = skip_alpha(*str);

	if (!my_isnumber(*component->end)) {
		switch (classify_keyword(component->start, component->end, flags)) {
		case KEYWORD_UNKNOWN:
			component->metaorder = METAORDER_LETTER_SUFFIX;
			break;
		case KEYWORD_PRE_RELEASE:
			component->metaorder = METAORDER_PRE_RELEASE;
			break;
		case KEYWORD_POST_RELEASE:
			component->metaorder = METAORDER_POST_RELEASE;
			break;
		}

		*str = component->end;
		return 2;
	}
}
```

```c
// the numeric bit
component->start = *str = skip_zeroes(*str);
component->end = *str = skip_number(*str);

if (component->start == component->end) {
	component->metaorder = METAORDER_ZERO;
} else {
	component->metaorder = METAORDER_NONZERO;
}
```

You've seen most of these helper functions before, although `my_memcasecmp` is new, as is the `my_tolower` it depends on:
```c
static inline int my_memcasecmp(const char* a, const char* b, size_t len) {
	while (len != 0) {
		unsigned char ua = my_tolower(*a);
		unsigned char ub = my_tolower(*b);

		if (ua != ub)
			return ua - ub;

		a++;
		b++;
		len--;
	}

	return 0;
}

static inline char my_tolower(char c) {
	if (c >= 'A' && c <= 'Z')
		return c - 'A' + 'a';
	else
		return c;
}
```

That's 6 lines to describe it, 5 lines to implement it in Ruby, and 65 lines in the C implementation.

Ruby benefits from syntax here, as [trailing conditionals](https://docs.ruby-lang.org/en/master/syntax/control_expressions_rdoc.html#label-Modifier+if+and+unless) allow for individual rank checks to be condensed to one line. The use of [`Enumerable.include?`](https://docs.ruby-lang.org/en/master/Enumerable.html#method-i-include-3F) lets me check for the prescence of multiple strings compactly, while [`String.start_with?`](https://docs.ruby-lang.org/en/master/String.html#method-i-start_with-3F) as a standard library means I don't have to bring my own, which the C implementation does in the form of `my_memcasecmp` and the accompanying `my_tolower`. [`Integer.zero?`](https://docs.ruby-lang.org/en/master/Integer.html#method-i-zero-3F) and [`Numeric.nonzero?`](https://docs.ruby-lang.org/en/master/Numeric.html#method-i-nonzero-3F) are nice, but they aren't that much better than how the C implementation does it. However, having the components actually split into an array lets me handle `LETTER_SUFFIX` very nicely, as I can just pass in the previous and next components when determining rank, and once again bundle everything onto one line. It's not the prettiest code I've ever written, but it gets the job done, and is adequately documented and understandable in the actual source code.

You might notice that the C implementation is catching up, as the Ruby implementation has gone from being 49x more compact in the first step down to only 13x more compact in the second part of the algorithm. Let's see if that holds as we move on to the third step of the algorithm, shall we?

### Step 3

```
Versions are compared component-wise.

- Ranks are compared first, and are ordered the same way as introduced above (PRE_RELEASE < ZERO < POST_RELEASE < NONZERO < LETTER_SUFFIX).
- Alphabetic components are compared by case insensitively comparing their first letters (choice of this behavior explained below).
- Numeric components are compared numerically.
```

To compare rank order, I store all the ranks in an array and just compare the numeric values of their indices. The only other notable thing here is `Enumerable.none?`, which I use to avoid having to type out `component.match?(/[^[:alpha:]]/)` twice:
```ruby
# Compare the rank of both components, and return the result unless they have the same rank.
rank_comparison = RANK_ORDERING.index(self.rank) <=> RANK_ORDERING.index(other.rank)
return rank_comparison unless rank_comparison.zero?

# If both components are alphabetic, return the case-insensitive comparison of their first letters.
return self.component[0].downcase <=> other.component[0].downcase if [self, other].none? { _1.component.match?(/[^[:alpha:]]/) }

# If we are still here, return the integer comparison of both components.
return self.component.to_i <=> other.component.to_i
```

We see a lot of the same beats in the C implementation, with the only particularly notable bit being the use of an enum instead of an array to determine rank order:
```c
/* metaorder has highest priority */
if (u1->metaorder < u2->metaorder) {
	return -1;
}
if (u1->metaorder > u2->metaorder) {
	return 1;
}

/* empty strings come before everything */
int u1_is_empty = u1->start == u1->end;
int u2_is_empty = u2->start == u2->end;

if (u1_is_empty && u2_is_empty) {
	return 0;
}
if (u1_is_empty) {
	return -1;
}
if (u2_is_empty) {
	return 1;
}

/* alpha come before numbers  */
int u1_is_alpha = my_isalpha(*u1->start);
int u2_is_alpha = my_isalpha(*u2->start);

if (u1_is_alpha && u2_is_alpha) {
	if (my_tolower(*u1->start) < my_tolower(*u2->start)) {
		return -1;
	}
	if (my_tolower(*u1->start) > my_tolower(*u2->start)) {
		return 1;
	}
	return 0;
}
if (u1_is_alpha) {
	return -1;
}
if (u2_is_alpha) {
	return 1;
}

/* numeric comparison (note that leading zeroes are already trimmed here) */
if (u1->end - u1->start < u2->end - u2->start) {
	return -1;
}
if (u1->end - u1->start > u2->end - u2->start) {
	return 1;
}

int res = memcmp(u1->start, u2->start, u1->end - u1->start);
if (res < 0) {
	return -1;
}
if (res > 0) {
	return 1;
}
return 0;
```

That's 4 lines to describe it, 4 lines to implement it in Ruby, and 48 lines in the C implementation.

Ruby benefits strongly here from the [`<=>` operator](https://docs.ruby-lang.org/en/master/Integer.html#method-i-3C-3D-3E), which, along with trailing conditionals, lets me again implement most steps of the algorithm in one or two lines.

For those paying attention, the C implementation continues to close the gap, with Ruby now only being 12x more compact. Regardless, that's just about it for this comparison, as we've covered the majority of the algorithm, with the remaining differences being down to boring implementation details.

## Conclusion

`libversion` takes 30 lines to describe, 303 lines to implement in C, and 65 to implement in Ruby. That leaves Ruby as being a little over 4.5x more compact than C, which isn't too shabby.

Now, of course, you may disagree with my implicit conclusion that Ruby enables more compact code than C. For example, you might point out that this is just one example, and that there are plenty of cases where good, compact C exists. This is true! I'm sure that somewhere out there exists some high-quality C code. I personally have yet to see any, but that's beside the point.

Obviously, dear reader, you are the best C/Rust/Raku developer in the world, and you could implement `libversion` in 60, 30, 10, or even 2 lines. I have no doubt about that. Unfortunately, not all code is written by you. In fact, the majority probably isn't.

This leads nicely into my next point, which is that I am not the best Ruby developer in the world. Not even close. I only have a few years of experience, and I'm learning new things all the time. Just a few days ago, I figured out a better way to iterate over a zipped array, that saves a couple lines in my implementation of `libversion`. The point is, when I compare the C and Ruby implementations of `libversion`, I'm comparing average C code with average Ruby code. Statistically, most comparisons will be of average examples.

Is the best Ruby code >4.5x more compact than the best C code? I have no idea. But when it comes to average, real-world code, Ruby does tend to be more compact than C, by virtue of having more expressive syntax and a powerful standard library.

Of course, you might also point out that compact code isn't necessarily better or more readable code. This is also true! Explicitly focusing on minimising lines of code can lead to some truly terrible code being created. Although, when rewriting `libversion` in Ruby, my goal wasn't to make something more compact than the C implementation, it just so happened to turn out that way. Furthermore, the Ruby implementation is thoroughly documented, so even if you can't read Ruby, you can probably read English. If you can't read English, there's never been a better time to learn Ruby.

Finally, performance. As everyone knows, C is a compiled, low-level language, while Ruby is a high-level, interpreted language, and therefore Ruby is slower.

I could, of course, drag out the quote about how premature optimisation is the root of all evil, and consider that not all software needs to be as fast as possible at the cost of everything else. I could also point out how `libversion` isn't the bottleneck, as for both Repology and Chromebrew what actually takes time is the network calls required to get the versions to compare.

Or, I could just point out that the Ruby implementation is faster than the C implementation:

```
 ruby 3.3.8 (2025-04-09 revision b200bad6cd) +RJIT [x86_64-linux-gnu]
Warming up --------------------------------------
    C implementation     4.782B i/100ms
 Ruby implementation     8.435B i/100ms
Calculating -------------------------------------
    C implementation    324.351B (±16.7%) i/s    (0.00 ns/i) -      3.137T in   9.997187s
 Ruby implementation    598.690B (±14.2%) i/s    (0.00 ns/i) -      5.862T in  10.006527s

Comparison:
 Ruby implementation: 598689997949.0 i/s
    C implementation: 324350915922.9 i/s - 1.85x  slower
```

That will be all.
