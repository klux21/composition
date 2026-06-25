<p align="center">
  <img src="composition.svg" width="300">
</p>

<h2>The untyped, structured, hierarchical, user friendly data composition format</h2>


Hierarchical data compositions are a modern untyped configuration file format.
They are using a minimal set of structural elements for creating structured data.

They are considering the fact that many people don't need more than a simple list of
some name and value pairs for the handful of program parameters that even common
INI-files deem an overkill for that.

But sometimes those need to extend their configurations by a simple hierarchical
structure for describing the subelements of some new and or more complex entries.
Adding those subentries to a string and parsing that independently after reading it
from the configuration file would by the most common way to solve this.
Of course this comes with a flaw because it is very inflexible to extend and may
even become a nightmare if the entries in such a string have subelements as well
which require an escaping of special characters within their argument strings.

For solving this problem a different kind of entry is required that's not a string
but holds subconfigurations only. One way to solve that would be to using JSON or XML
but those require to a switch that configuration to their rigid and quite different
syntax and are turning that little configuration file from before into an eye sore.

But all you really need is a different kind of string that holds the configuration
of the subparameters for being parsed independently. That's where this slightly
different data composition format comes from which is referred as a 'composition'
from now.

Every composition consists of an ordered collection of entries.
The entries consist of a name that is optionally followed by either an argument
or a nested composition document that the name refers to.
Composition documents can be divided into sections for an improved readability
and a limited compatibility with existing INI-based configurations.

Those compositions keep the simplicity and readability of INI but add:

 -   hierarchical blocks using opening and closing braces `{` `}`
 -   optional sections
 -   quoted and unquoted strings
 -   robust comment syntax
 -   the option of a zero-copy-friendly parsing

Compositions have no type systems; values are always strings or subdocuments.
Compositions are designed for:

 -   configuration files
 -   embedded systems
 -   lightweight data serialization
 -   small text databases
 -   text-based protocols
 -   DOM-style structured data

They are intentionally minimal, deterministic, and easy to parse in C and other languages.

But why compositions?

 - Classic INI is easy to read and write but not hierarchical and usually line-separated.
 - TOML is similar to INI but adds type restrictions.
 - JSON is hierarchical but verbose, rigid, and lacks comments.
 - YAML is flexible but fragile.
 - XML  is verbose and not great to read for humans or machines.
 - And compositions? Well, those are quite a different kind of music that's for grown ups.

Structured compositions are easy to create and trivial to use. They don't require a
college degree or big manuals to understand the syntax. They may contain comments because
they are not JSON and comments should be used where those are helpful.

But let's start with a sample of what a composition document may look like.

```
[server]

host = "localhost"
port = 8080

tls = {
         enabled
         certificate = "/etc/certs/server.pem"

         [ciphers]

         #*
            comment block
         *#

         accept = {
                     TLS_AES_128_CCM_8_SHA256
                     TLS_CHACHA20_POLY1305_SHA256
                     TLS_AES_128_GCM_SHA256
                  }
      }

# line comment
```
Well, that looks a bit like INI except that there is that tls block that contains
a composition of entries which look more or less like an INI file as well.
That is basically all that structured data compositions need.

Structured compositions have a very minimal syntax; a document consists of three
types of elements:

  - Entries which consists of a name string that can be followed by an equality sign
    and an argument value.
  - Blocks of subdocuments, as a special type of entry, where the argument is an
    independent composition document enclosed in curly braces.
  - Sections, which divide a data composition into parts and consist of a section name
    enclosed in square brackets, followed by the entries which are belonging to that section.

Blocks can contain sections, and sections can contain blocks as well.
Besides that, there are two types of comments:

  - Line comments that begin with a hash `#` or a semicolon `;` and end at the end
    of the line.
  - Block comments start with a `#*` or `;*` sequence and end with the start sequence in
    reversed order which is `*#` or `*;`.

Of course there is a little bit more. For a more flexible usage of structured compositions,
line feeds are used for terminating line comments but otherwise they are entry-separating
whitespace only. Therefore a single line would be enough for a composition document like
before

```
[server] host="localhost" port=8080 tls={ enabled certificate="/etc/certs/server.pem" [ciphers] #* comment block *# accept={ TLS_AES_128_CCM_8_SHA256  TLS_CHACHA20_POLY1305_SHA256  TLS_AES_128_GCM_SHA256 }}} # line comment
```
This may seem a little bit unconventional first but allows to treat every simple name
value pair argument list as a composition. Of course strings that contain whitespaces or
special characters require a pair of enclosing quotes for all of their parts that contain
those.

Equals signs `=` before curly braces aren't really required for the syntax and optional.
The support of sections within compositions ensures compatibility with most existing INI files but
it's also fine to omit sections completely.
That way the configuration file above becomes even more trivial:

```
server {
   host = localhost
   port = 8080

   tls {
      enabled
      certificate = "/etc/certs/server.pem"

      ciphers {
         #* comment block *#
         accept { TLS_AES_128_CCM_8_SHA256  TLS_CHACHA20_POLY1305_SHA256  TLS_AES_128_GCM_SHA256 }
      }
   }
}
# line comment
```

That's not XML, JSON or INI but a very lightweight and well structured configuration.
However, in case of more complex configurations sections may help to improve the readability.

But what's with multi-line strings as in TOML? Well, because line-feeds are treated as whitespace
strings may contain line-feeds and are allowed to span over as many lines as you want.

For parsing composition documents the application needs to know what entry names it expects,
what types the arguments are of, where the entries are located within document structure,
and whether it is OK to ignore unexpected or missing entries. It's up to the application whether
the entries are allowed to be in a random order or whether they need to match a predefined
sequence. But this is neither for the users nor the application fundamentally different from
other data formats and always a decision of the developers only.

Compositions have no type semantics, no implicit conversions and no magic and are zero-copy friendly.
 
So what does it take to parse something like that in C?

- Copy the document into a character buffer in memory.
  Conforming composition parsers should stop at a `'\0'` in documents, so it's a good idea to
  terminate the data like a C string.
- The parser object can be as simple as character pointer that iterates the buffer.
- Skip blanks, comments and line feeds which are just entry separating whitespace and iterate
  the entries until the end of the enclosing block or the end of the document.
- If a new section starts, the content after the section header belongs to this section.
  Continue the iteration after parsing the section header for the section name.
- The content of sections and blocks can be returned as unquoted and unescaped string data or
  just a pointer to the beginning of that data.
- The composition parser should provide helper functions for removing the quotes from strings
  and optionally replacing the predefined escape sequences within strings.
- Applications need to check the entry names and to convert the argument strings into their
  internal data types; e.g. if an argument holds the value of a double they may call strtod()
  or they call strptime() if the argument refers to a time value.

That's nearly all of the magic that parsing composition documents requires.
Once you have a parser, composition documents are very trivial to handle and a very powerful tool.

Usually it's not the parsing of the document structure but the conversion of the strings
to integers, floats and times that consumes most of the CPU time.
If this becomes a problem, or if your compiler doesn't support hexadecimal floats or the
binary and octal prefixes 0b and 0o already, try:

 https://github.com/klux21/str2num

The following project may help with time values

 https://github.com/klux21/limitless_times

One of the greatest advantages of composition documents is their very lenient syntax.
The format does not have many special elements. A simple list of whitespace or
line feed separated numbers or words and many INI files are already are valid composition
documents. Because of this the most common file extensions for composition documents are
.ini or .cfg and rarely .cof that indicates a composition file.

Of course most people have very different ideas how their very own compositions should be.
To reduce deviations between parsers, there is an initial draft of a tiny composition
standard that conforming parsers should match

 https://github.com/klux21/composition/blob/main/composition_standard.txt

If a parser intentionally deviates from that, it should document the differences.
E.g. if it treats comma as whitespace and colons as equals signs for parsing JSON files
and allowing common composition comments in those. Of course such a parser would have trouble
with composition documents that are using commas and colons outside of embracing quotes so the
users need to be aware of that. (However, it would be a better idea to implement a special
conversion function for JSON data in that case.)

Of course it's more likely that some people dislike some of the features and prefere a subset
of that tiny format only. That's OK, but sectionless, commentless, unescaped, unquoted or
non-hierarchical only data composition parsers should be refered as such to prevent confusions.

But how fast is it? Parsing the following document zero-copy on a Ryzen 3900 took about
400ns for finding all entries and 150ns for converting the eight numbers to either int64_t
or double using str2i64_r str2d_r of klux21/str2num:
```
testname = zero_copy_tests
inttests = { ib=0b1111 io=0o1234567 id=000056789 ix=0xabcd987 }
floattests { fb=0b11.11e100 fo=0o1234.56e10 fd=1.2345e64 fx=0xabc.defp10 }
```
The end of the subdocuments were searched before looking for the entries because of the
treatment of subdocuments as strings. It could be faster without that.
If all entries are copied into newly allocated strings before reading them than it's half
as fast only. That's a common thing if removing quotes and replacing escapes requires a
buffer for the modfied data.

The test project with the benchmark and the little parser for C and C++ can be found at

https://github.com/klux21/composition_parser


| Format/Language | Hex-Float Support (`0x...p...`) | NaN/Inf Support | Line Comments | Block Comments | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **C / C++** | ✓ Yes (since C99/C++17) | ✓ Yes | ✓ Yes (`//`) | ✓ Yes (`/* */`) | Standard via `%a` / `std::hexfloat`. |
| **Java** | ✓ Yes (Parser) | ✓ Yes | ✓ Yes (`//`) | ✓ Yes (`/* */`) | `Double.parseDouble("0x1.ap2")` works, but not in JSON/HOCON files.  |
| **Python** | ✓ Yes | ✓ Yes | ✓ Yes (`#`) | :x: **No** | `float.fromhex()` and literal support. |
| **TOML** | :x: No | ✓ Yes | ✓ Yes (`#`) | :x: **No** | Supports Hex-Ints, but *no* Hex-Floats (uses high-precision decimal). |
| **YAML** | :x: No (Standard) | ✓ Yes | ✓ Yes (`#`) | :x: **No** | No native Hex-Float syntax, but extensible via tags. |
| **HOCON** | :x: **No** | :x: **No** | ✓ Yes (`#`, `//`) | :x: **No** | Strictly limited to decimal JSON numbers.  |
| **JSON** | :x: **No** | :x: **No** | :x: **No** | :x: **No** | Strictly limited to decimal JSON numbers; no comments allowed by spec.  |
| **Composition** | ✓ Yes | ✓ Yes | ✓ Yes (`#`, `;`) | ✓ Yes (`#* *#`,`;* *;`) | Hex-Floats are industrial standard since tens of years and only strings in configurations. There is no need for prohibiting them. |

That's all about it. Once there are errors or problems, file an issue at:

https://github.com/klux21/composition/issues
