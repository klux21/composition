<p align="center">
  <img src="composition.svg" width="300">
</p>

<h2>COMPOSITION - An untyped, structured, hierarchical data format</h2>


COMPOSITION is a modern untyped configuration file format with hierarchical elements.

A composition document consists of an ordered collection of entries.
Each entry consists of a name optionally followed by either an argument or a nested
composition document of that name.
Composition documents may be divided into sections for improved readability and
compatibility with existing INI-based configurations.

It keeps the simplicity and readability of INI but adds:

 -   hierarchical blocks using opening and closing braces { }
 -   optional sections
 -   quoted and unquoted strings
 -   robust comment syntax
 -   zero-copy-friendly parsing

COMPOSITION has no type systems; values are always strings or subdocuments.

COMPOSITION is designed for:

 -   configuration files
 -   embedded systems
 -   lightweight data serialization
 -   small text databases
 -   text-based protocols
 -   DOM-style structured data

It is intentionally minimal, deterministic, and easy to parse in C and other languages.

Why COMPOSITION?

 - Classic INI is easy to read and write but not hierarchical and usually line-separated.
 - TOML is similar to INI but adds type restrictions.
 - JSON is hierarchical but verbose, rigid, and lacks comments.
 - YAML is flexible but fragile.
 - XML  is verbose and not great to read for humans or machines.
 - And COMPOSITION? That's a different kind of music.

Composition documents are easy to read and trivial to use. They are easy to understand
and don't require a college degree or big manuals to understand the syntax.
Composition documents provide data in an easy and structured way.
COMPOSITION supports comments because it's not JSON and comments can be added where
they are helpful.

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
       block comment
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
That's already what composition documents are.

COMPOSITION defines only a minimal syntax; a document consists of three
types of elements:

  - Entries which consists of a name string that can be followed by an equality sign
    and an argument value string.
  - Blocks, a special type of entry, whose argument is an independent composition
    subdocument enclosed in curly braces.
  - Sections, which divide a composition into parts and consist of a section name
    enclosed in square brackets, followed by the entries belonging to that section.

Blocks can contain sections, and sections can contain blocks as well.
Besides that, there are two types of comments:

  - Line comments that begin with a hash '#' or a semicolon ';' and end at the end
    of the line.
  - Block comments start with '#*' or ';*' and end with their start sequence in
    reversed order which is '*#' or '*;'.

Of course there is a little bit more. In composition documents, line feeds terminate
line comments and otherwise they are entry‑separating whitespace only.
A single line for the composition document before is enough e.g.

```
[server] host="localhost" port=8080 tls = { enabled certificate="/etc/certs/server.pem" [ciphers] #* comment block *# accept = { TLS_AES_128_CCM_8_SHA256  TLS_CHACHA20_POLY1305_SHA256  TLS_AES_128_GCM_SHA256 }}} # line comment
```
Equals signs ('=') before curly braces are optional and a question of personal taste.
Strings that contain whitespaces or special characters require a pair of
enclosing quotes for their parts that contain those.
The support of sections ensures compatibility with most existing INI files but
it's also fine to omit sections. That way configuration file above becomes
a bit more trivial, e.g.

```
server
{
   host = localhost
   port = 8080

   tls
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

That's not XML, JSON or INI but a very lightweight and well structured configuration.
However, in case of more complex documents sections may improve the readability.

For parsing composition documents the application needs to know what entry names it expects,
what types the arguments are, where those are located within document structure, and whether
it is OK to ignore unexpected or missing entries. It's also up to the application whether
the entries are allowed to be in a random order or whether they need to match a predefined sequence.
But this is neither for the users nor the application fundamentally different from other
data formats and always a decision of the developers either.

COMPOSITION has no types, no implicit conversions and no magic and is zero-copy friendly.
 
So what does it take to parse something like that in C?

- Copy the document into a character buffer in memory.
  Conforming composition parsers should stop at a '\0' in documents, so it's a good idea to
  terminate the data like a C string.
- The parser object can be as simple as character pointer that iterates the buffer.
- Skip blanks, comments and line feeds which are just entry separating whitespace and iterate
  the entries until the end of the enclosing block or the end of the document.
- If a new section starts, the content after the section header belongs to this section.
  Continue the iteration after parsing the section header for the section name.
- The content of sections and blocks can be returned as unquoted and unescaped string data or
  just a pointer to the beginning of that data.
- A COMPOSITION parser should provide helper functions for removing the quotes from strings
  and optionally replacing the predefined escape sequences within strings.
- Applications need to check the entry names and to convert the argument strings into their
  internal data types; e.g. if an argument holds the value of a double they may call strtod().

That is nearly all of the magic that parsing composition documents requires.
Once you have a good parser, composition documents are trivial to handle and a very
powerful tool.

Usually it's not the parsing of the document structure but the conversion of the strings
to integers, floats and times that consumes most of the CPU time.
If this becomes a problem, or if your compiler doesn't support hexadecimal floats or the
binary and octal prefixes 0b and 0o, try: 

 https://github.com/klux21/str2num

The following project may help with time values

 https://github.com/klux21/limitless_times

One of the greatest advantages of composition documents is their very lenient syntax.
The format does not have many special elements. A simple list of whitespace or
line feed separated numbers or words and many INI files are already are valid composition
documents. Because of this the most common file extensions for composition documents are
.ini or .cfg and rarely .cof that indicates a COMPOSITION file.

Of course most people have very different ideas how their compositions should be.
To reduce deviations between parsers, there is an initial draft of the COMPOSITION standard

 https://github.com/klux21/composition/blob/main/composition_standard.txt

If a parser intentionally deviates from that, it should document the differences.
E.g. it may treat of comma as whitespace and colons as equals signs for parsing JSON files
and allowing COMPOSITION comments in those. Of course such a parser would have trouble with
COMPOSITION documents that are using commas and colons outside of embracing quotes so the
users need to be aware of that. (Of course it would be better to implement a conversion
function for JSON data in that case.)

Once there are errors or problems, please file an issue at:

https://github.com/klux21/composition/issues
