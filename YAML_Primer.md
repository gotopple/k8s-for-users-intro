# YAML Primer

Not everyone is familiar with Yet Another Markup Language (YAML) and that can be a hurdle to adopting Kubernetes. While you can use either JSON or YAML to represent resources, most of the available examples use YAML. The good news is that it is rare to see people use exotic YAML features. Most people stick to the basics. What follows is a brief primer of common YAML parts.

The full YAML 1.2 specification is available [here](http://yaml.org/spec/1.2/2009-07-21/spec.html) and is really quite readable.

## YAML Documents and Scope

YAML is used to describe structured documents which are made up of structures, lists, maps, and scalar values. Those features are defined as continuous blocks, where substructures are defined with nested block definitions in a stack style that will feel familiar to most high-level language programmers.

The default scope of a document is a single file or stream. However a single document or stream can contain multiple documents. You can indicate the end of a previous document and the beginning of a new document with the `---` token.

## Comments

Any line can contain a comment at the end of the line. Comments are marked by a " #" (space+hash) character sequence. Any characters that follow until the end of the line are ignored by the parser.

**Empty lines are just fine.**

## Maps, Lists, and Scalars

YAML uses three types of data and two styles, "block" and "flow." Flow collections are specified similar to collection literals in JavaScript and other languages. The block style is more common. 

**Maps** are defined by a set of unique properties in the form of key-value pairs where the pairs are colon+space delimited. While property names must be string values, property values can be any of the YAML data types except documents. A single structure cannot have multiple definitions for the same property. Example:

```
image: "alpine"
command: echo hello world
```

This document contains a single map with two properties: `image` and `command`. The `image` property has a scalar string value, `alpine`. The command property has a scalar string value `echo hello world`. A **scalar** is a single value. The example above demonstrates two of the three flow scalar styles. 

The value for `image` is specified in "double-quote style" which is capable of expressing arbitrary strings, by using “\” escape sequences. Most programmers are familiar with this string style.

The value for `command` is written in "plain style." The plain (unquoted) style has no identifying indicators and provides no form of escaping. It is therefore the most readable, most limited and most context sensitive style. There are a bunch of rules for plain style scalars. Plain scalars:

* must not be empty
* must not contain leading or trailing white space characters
* must not begin with an indicator character (like - or :) where doing so would cause an ambiguity
* must never contain ": " or " #" character combinations

**Lists** (or block sequences) are series of nodes where each element is denoted by a leading “-” indicator. For example:

```
- item 1
- item 2
- item 3
- # an empty item
- item 4
```

## Whitespace

YAML uses indentation to indicate content scope. There are a few rules:

* Only spaces can be used for indentation
* The amount of indentation does not matter as long as...
* All peer elements (in the same scope) have the same amount of indentation
* and child elements are further indented

These three documents are equivalent:

```
top-level:
   second-level:          # three spaces
     third-level:         # two more spaces
      - "list item"       # single additional indent on items in this list
     another-third-level: # a third-level peer with the same two spaces 
           fourth-level: "string scalar" # 6 more spaces 
   another-second-level:  # a 2nd level peer with three spaces
               - a list item # list items in this scope have 15 total leading spaces
               - a peer item
---
# every scope level adds exactly 1 space
top-level:
 second-level:
  third-level:
   - "list item"
  another-third-level:
   fourth-level: "string scalar"
 another-second-level:
  - a list item
  - a peer item
---
# every scope level adds exactly 2 spaces
top-level:
  second-level:
    third-level:
      - "list item"
    another-third-level:
      fourth-level: "string scalar"
  another-second-level:
    - a list item
    - a peer item
```
