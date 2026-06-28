<p align="center">
  <img src="composition.svg" width="300">
</p>

<h2>The untyped, structured, hierarchical, user friendly data composition format</h2>

The data composition format is one of the fastest and most flexible hierarchical text
data formats. It consists of strings, structural markers, and comments only, yet it
can give highly complex configurations a remarkable readability.

It considers the fact that many people need nothing more than a simple list of
name–value pairs for a handful of program parameters - something for which even common
INI files usually seem like an overkill.

But sometimes such trivial configurations need a more hierarchical structure to
describe the subelements of more complex entries.

One way to achieve this would be to switch to JSON or XML, but both come with rigid
syntax rules and would turn that simple configuration file into an eyesore.

The most common workaround would be to pack the subentries into a single string and
parse that string independently after reading it from the configuration file.
However, this comes with a serious flaw: such a hierarchical structure is hard
to extend and may even become a nightmare of escaping if the entries within such a
string have subelements as well which require an escaping of some special characters
and the quotes of their arguments strings.

Of course, all you really need is a different kind of string — one that holds the
configuration of the subparameters, can be parsed independently, and must never be
unescaped because it's an independent subdocument. That's where this slightly
different data composition format comes from which is referred as a 'composition'
from now.

The composition format consists of an ordered collection of entries. The entries
consist of a name that is optionally followed by either an argument or a nested
composition document that the name refers to.

Composition documents can be divided into sections for an improved readability
and a limited compatibility with existing INI-based configurations.
Those documents keep the simplicity and readability of INI but add:

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
E.g. if it supports C and C++ comments. Such a parser would have trouble with URIs or paths
with wildcards outside of embracing quotes. For this the users need to be aware of that and
not expecting it to handle data composition files according to the standard.

Of course it's more likely that some people dislike some of the features and prefere a subset
of that tiny format only. That's OK, but sectionless, commentless, unescaped, unquoted or
non-hierarchical only data composition parsers should be refered as such to prevent confusions.

But how fast is it? Parsing the following document zero-copy on a Ryzen 3900 took about
350ns for finding all entries and 150ns for converting the numbers to either int64_t
or double using str2i64_r str2d_r of klux21/str2num:
```
testname = zero_copy_tests
inttests = { ib=0b1111 io=0o1234567 id=000056789 ix=0xabcd987 }
floattests { fb=0b11.11e100 fo=0o1234.56e10 fd=1.2345e64 fx=0xabc.defp10 }
```
The end of the subdocuments were determined before looking for the entries because of the
treatment of the subdocuments as strings. It could be faster without that.
If all entries are copied into newly allocated strings before reading them than it's half
as fast only. That may be required for removing quotes and replacing escapes.

The test project of that measurement and the little parser for C and C++ can be found at

https://github.com/klux21/composition_parser

That's all about it. Once there are errors or problems, file an issue at:

https://github.com/klux21/composition/issues

