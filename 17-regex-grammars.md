# Reading 17: Regular Expressions & Grammars

## Grammars

To describe a string of symbols, whether they are bytes, characters, or some other kind of symbol drawn from a fixed set, we use a compact representation called a grammar.

A *grammar* defines a set of strings. Suppose we want to write a grammar that represents URLs. Our grammar for URLs will specify the set of strings that are legal URLs in the HTTP protocol.

The literal strings in a grammar are called *terminals*. They’re called terminals because they can’t be expanded any further. We generally write terminals in quotes, like '`http'` or `':'`.

A grammar is described by a set of *productions*, where each production defines a *nonterminal*. You can think of a nonterminal like a variable that stands for a set of strings, and the production as the definition of that variable in terms of other variables (nonterminals), operators, and constants (terminals). Nonterminals are internal nodes of the tree representing a string.

A production in a grammar has the form

```
nonterminal ::= expression of terminals, nonterminals, and operators 
```

One of the nonterminals of the grammar is designated as the *root*. The set of strings that the grammar recognizes are the ones that match the root nonterminal. This nonterminal is sometimes called root or start or even just S, but in the grammars below we will typically choose more readable names for the root, like url, html, and markdown.


## Regular expressions

A regular grammar has a special property: by substituting every nonterminal (except the root one) with its righthand side, you can reduce it down to a single production for the root, with only terminals and operators on the right-hand side.

Our URL grammar is regular. By replacing nonterminals with their productions, it can be reduced to a single expression:

```
url ::= 'http://' ([a-z]+ '.')+ [a-z]+ (':' [0-9]+)? '/' 
```

The Markdown grammar is also regular:

```
markdown ::= ([^_]* | '_' [^_]* '_' )*
```

But our HTML grammar can’t be reduced completely. By substituting righthand sides for nonterminals, you can eventually reduce it to something like this:

```
html ::= ( [^<>]* | '<i>' html '</i>' )*
```

…but the recursive use of html on the righthand side can’t be eliminated, and can’t be simply replaced by a repetition operator either. So the HTML grammar is not regular.

The reduced expression of terminals and operators can be written in an even more compact form, called a *regular expression*. A regular expression does away with the quotes around the terminals, and the spaces between terminals and operators, so that it consists just of terminal characters, parentheses for grouping, and operator characters. For example, the regular expression for our markdown format is just

```
([^_]*|_[^_]*_)*
```

Regular expressions are also called regexes for short. A regex is far less readable than the original grammar, because it lacks the nonterminal names that documented the meaning of each subexpression. But many programming languages have library support for regexes (and not for grammars), and regexes are much faster to match than a grammar.

The regex syntax commonly implemented in programming language libraries has a few more special characters, in addition to the ones we used above in grammars. Here are some common useful ones:

```
.   // matches any single character (but sometimes excluding newline, depending on the regex library)

\d  // matches any digit, same as [0-9]
\s  // matches any whitespace character, including space, tab, newline
\w  // matches any word character including underscore, same as [a-zA-Z_0-9]
```

Backslash is also used to “escape” an operator or special character so that it matches literally. Here are some of the common special characters that you need to escape:

```
\.  \(  \)  \*  \+  \|  \[  \]  \\
```

Using backslashes is important whenever there are terminal characters that would be confused with special characters. Because our `url` regular expression has `.` in it as a terminal, we need to use a backslash to escape it:

```
http://([a-z]+\.)+[a-z]+(:[0-9]+)?/
```

Another way to escape a special character is to wrap it in character-class brackets. Instead of `\.`, we could also write `[.]` to match just the literal `.` character. Inside character-class brackets, most special characters lose their special meaning, and are simply treated literally. But characters that are special to the character-class syntax, like `[`, `]`, `^`, `-`, and `\`, still need to be escaped by a backslash to be used literally.

### Using regular expressions in practice

n Java, you can use regexes for manipulating strings (see `String.split,` `String.matches`, `java.util.regex.Pattern`). They’re built-in as a first-class feature of modern scripting languages like Python, Ruby, and JavaScript, and you can use them in many text editors for find and replace. Regular expressions are your friend! Most of the time. Here are some examples.

Replace all runs of spaces in a string s with a single space:

```jav
String singleSpacedString = s.replaceAll(" +", " ");
```

Match a URL:

```java
if (s.matches("http://([a-z]+\\.)+[a-z]+(:[0-9]+)?/")) {
    // then s is a url
}
```

Notice the backslashes in the example above. We want to match a literal period `.`, so we have to first escape it as `\.` to protect it from being interpreted as the regex match-any-character operator, and then we have to further escape it as `\\.` to protect the backslash from being interpreted as a Java string escape character. The frequent necessity for double-backslash escapes makes regexes still less readable.

Extract parts of a date like `"2020-03-18"`:

```java
String s = "2020-03-18";
Pattern regex = Pattern.compile("(?<year>\\d{4})-(?<month>\\d{2})-(?<day>\\d{2})");
Matcher m = regex.matcher(s);
if (m.matches()) {
    String year = m.group("year");
    String month = m.group("month");
    String day = m.group("day");
    // Matcher.group(name) returns the part of s that matched (?<name>...)
}
```

This example uses *named capturing groups* like `(?<year>...)` to extract parts of the matched string and assign them names. The `(?<name>...)` syntax matches the regex ... inside the parentheses, and then assigns name to the string that match. Note that ? here does not mean 0 or 1 repetition. In this context, right after an open parenthesis, the ? signals that these parentheses have special meaning, not just grouping.

Named capturing groups can be retrieved by the `group()` method after a successful match. If this regex were matched against `"2025-03-18"`, for example, then `m.group("year")` would return `"2025"`, `m.group("month")` would return `"03"`, and `m.group("day")` would return `"18"`.
