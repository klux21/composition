<p align="center">
  <img src="composition.svg" width="300">
</p>

<h1>COMPOSITION - An untyped structured hierarchic data format</h1>


COMPOSITION is a modern untyped configuration file format with hierarchical elements.

A composition document consists of an ordered collection of entries.
Each entry consists of a name and optionally either an argument or a nested composition
that the name referes to.
Compositions documents may be divided into sections for improved readability and
compatibility with existing INI-based configurations.

It keeps the simplicity and readability of INI but adds:

 -   hierarchic blocks using opening and closing braces { }
 -   optional sections
 -   quoted and unquoted strings
 -   robust comment syntax
 -   enables a zero-copy friendly parsing
 -   has no type system (values are always strings or composition)

COMPOSITION is designed for:

 -   configuration files
 -   embedded systems
 -   lightweight data serialization
 -   small text databases
 -   text-based protocols
 -   DOM-style structured data

It is intentionally minimal, deterministic, and easy to parse in C and other languages.

Why COMPOSITION?

 - Classic INI is easy to read and write but not hierarchical and usually line separated.
 - TOML is kind of INI with type restrictions.
 - JSON is hierarchical but noisy and rigid and lacks comments.
 - YAML is flexible but fragile.
 - XML is XML. Not great to read for humans nor machines.
 - And COMPOSITION? That's a different kind of music.

Composition documents are easy to read and trivial to use. They are easy to understand
and don't require a college degree nor big manuals for that.  COMPOSITION supports
comments because it's not JSON and comments should be added wherever those deem helpful.
Composition documents are for providing data in an easy and structured way.

But let's start with a sample what a composition document may look like.

```
[server]
host = "localhost"
port = 8080

tls = {
    enabled
    certificate = "/etc/certs/server.pem"

    [ciphers]
    #* comment
       block *#

    accept = {
               TLS_AES_128_CCM_8_SHA256
               TLS_CHACHA20_POLY1305_SHA256
               TLS_AES_128_GCM_SHA256
             }
}
# line comment
```
Well, that looks a bit like INI except that there is that ssl block that contains
a composition of entries which look more or less like an INI file as well.
That's already what composition documents are.

COMPOSITION defines a minimal syntax only and a document consists of three
types of optional elements

  - Entries which consists of a name string that can be followed by an equality sign
    and an argument value string.
  - Blocks as a special type of entries where the argument values consist of an
    independent composition subdocument within a pair of curly braces.
  - Sections which are dividing composition into independent parts and consist of a
    section name string within a pair of square braces.

Blocks can contain sections and sections blocks as well.
Beside of that there are two types of comments

  - Line comments that begin with a hash # or a semicolon ; and end at the end
    of the line.
  - Block comments which start with either a hash # or a semicolon ; followed
    by an asterisk * and ending with the start character sequence in reversed
    order.

Of course there is a little bit more. In composition documents line feeds are
terminating line comments and are nothing but entry separating whitespaces beside
of that. For this a single line for a that composition document before is enough
e.g.

```
[server] host="localhost" port=8080 ssl = { enabled certificate="/etc/certs/server.pem" [ciphers] #* comment block *# accept = { TLS_AES_128_CCM_8_SHA256  TLS_CHACHA20_POLY1305_SHA256  TLS_AES_128_GCM_SHA256 }}} # line comment
```
Equal signs before curly braces are optional and a question of personal taste.
Strings that contain whitespaces or special characters require a pair of
entangling quotes for all their parts that contain those.
The support of sections ensures compatibility with most existing INI files but
it's totally OK to skip the usage of sections.
That way configuration file above would become a bit more trivial, e.g.

```
server
{
   host = localhost
   port = 8080

   ssl
   {
      enabled
      certificate = "/etc/certs/server.pem"

      ciphers
      {
         #* comment block *#
         accept { TLS_AES_128_CCM_8_SHA256  TLS_CHACHA20_POLY1305_SHA256  TLS_AES_128_GCM_SHA256 }
      }
   }
}
```

That's not XML and not JSON nor INI but a very lightweight and structured data composition.
However, in case of more complex documents sections may help to improve the readability.

For parsing composition documents the application needs to know what entry names it expects,
what types the arguments are where those are located within document structure and whether
it is OK to ignore unexpected or missing entries.
It's also up to the application whether the entries are allowed to be in a random order or
wether they need to match a predefined sequence.
But this makes neither for the user nor the application a very big difference to other
data formats and is always a decision of the developers only.

COMPOSITION has no types, no implicit conversions and no magic and is zero-copy friendly.
 
So what does it take to parse something like that in C ?

- Copy the document to character buffer in memory.
  Conforming composition parsers should stop at a '\0' in documents so it's always a good
  idea to terminate the document data like a C string.
  The parser object can be as simple as just a character pointer that iterates that buffer.
- Skip blanks, comments and line feeds which are just entry separating whitespace and iterate
  the entries until the end of the entangling block or the end of the document.
- Once there starts a section than the content after the section header belongs to that
  section. Continue the iteration after parsing the section header and remembering it's name.
- The content of sections and blocks can be returned as unquoted and unescaped string data or
  just a pointer to the beginning of that data.
- A COMPOSITION parser should provide helper functions for removing the quotes from strings
  and optionally replacing the predefined escape sequences within strings.
- Applications need to check the entry names and to convert the argument strings into their
  internal data e.g. if the arguments is expected to be a double it they may call strtod().

That is nearly all of the magic that parsing composition documents requires.
As soon as you have a good parser composition documents are trivial to handle and a very
powerful tool.

Usually it's not the parsing of the document structure but the conversion of the strings
to integers, floats and times that consumes most of the CPU time.
Once that is a problem or if your compiler doesn't supports hexadecimal floats or the
prefixing of binary and octal integer values with 0b and 0o then you may try the following
project

 https://github.com/klux21/str2num

The following project may help with time values

 https://github.com/klux21/limitless_times

One of the greatest advantages of composition documents are their very lenient syntax.
The format itself has not much special elements. A simple list of some white space or
line feed separated numbers or words and many simple configuration files are already
valid composition documents and readable using a COMPOSITION parser.
Because of that the most common file extension of composition documents are just
.ini, .cfg and .cof that indicates a COMPOSITION file.

Of course most people have very different ideas how their compositions should be.
For preventing a lot of deviations of the parsers there exist a first little
standard draft of the COMPOSITION format

 https://github.com/klux21/composition/blob/main/composition_standard.txt

Once a parser intentionally deviates from that it should be documented for the users.
E.g. a parser that treats comma as blank and colon like an equality sign to enable
parsing of JSON files and enabling COMPOSITION comments within them.
Of course such a parser would have trouble with COMPOSITION documents that are using
commas and colons outside of embracing quotes so it's better to implement a separate
conversion function for JSON.

Once there are errors or problems which should be solved than just file an error
for that in

https://github.com/klux21/composition/issues
