Syntax
------

The grammar consists of a sequence of rules of the form:
```
rule_name: expression
```

Optionally, a type can be included right after the rule name,
which specifies the return type of the Python function
corresponding to the rule:
```
rule_name[return_type]: expression
```
If the return type is omitted, then the return type is `Any`.

### Grammar Expressions

##### `# comment`
Python-style comments.

##### `e1 e2`
Match e1, then match e2.
```
rule_name: first_rule second_rule
```

##### `e1 | e2`
Match e1 or e2.

The first alternative can also appear on the line after the rule
name for formatting purposes.  In that case, a | can also be used
before the first alternative, like so:
```
rule_name[return_type]:
    | first_alt
    | second_alt
```

##### `( e )`
Match e.
```
rule_name: (e)
```

A slightly more complex and useful example includes using the grouping
operator together with the repeat operators:
```
rule_name: (e1 e2)*
```

##### `[ e ] or e?`
Optionally match e.
```
rule_name: [e]
```

A more useful example includes defining that a trailing comma is optional:
```
rule_name: e (',' e)* [',']
```

##### `e*`
Match zero or more occurrences of e.
```
rule_name: (e1 e2)*
```

##### `e+`
Match one or more occurrences of e.
```
rule_name: (e1 e2)+
```
##### `s.e+`
Match one or more occurrences of e, separated by s. The generated parse tree
does not include the separator. This is identical to `(e (s e)*)`.
```
rule_name: ','.e+
```

##### `&e`
Succeed if e can be parsed, without consuming any input.

##### `!e`
Fail if e can be parsed, without consuming any input.

An example taken from `data/python.gram` specifies that a primary
consists of an atom, which is not followed by a `.` or a `(` or
a `[`:
```
primary: atom !'.' !'(' !'['
```

##### `~e`
Commit to the current alternative, even if it fails to parse.
```
rule_name: '(' ~ some_rule ')' | some_alt
```
In this example, if a left parenthesis is parsed, then the other
alternative won't be considered, even if some_rule or ')' fail
to be parsed.

##### `&&e`
Fail immediatly if e fails to parse by raising the exception built using the
`make_syntax_error` method.

This construct can help provides better error messages.


### Keywords

Keywords are identified in the grammar as quoted names. Single quotes `'def'`
are used to identify hard keywords i.e. keywords that are reserved in the grammar
and cannot be used for any other purpose. Double quotes `"match"` identify
soft keywords that act as keyword only in specific context. As a consequence a
rule matching `NAME` may match a soft keyword but never a hard keyword.

In some circonstances, it can desirable to match any soft keyword for those cases
one can use `SOFT_KEYWORD` that will expand to `"match" | "case"` if `match` and
`case` are the only two known soft keywords.

### Return Value

Optionally, an alternative can be followed by a so-called action
in curly-braces, which specifies the return value of the alternative:
```
rule_name[return_type]:
    | first_alt1 first_alt2 { first_alt1 }
    | second_alt1 second_alt2 { second_alt1 }
```
If the action is omitted, a list with all the parsed expressions gets returned.
However if the rule contains a single element it is returned as is without being
wrapped in a list. Rules allowing to match multiple items (`+` or `*`) always
return a list.

By default the parser does not track line number and col offset for production
each rule. If one desires to store the start line and offset and the end line
and offset of a rule, one can add `LOCATIONS` in the action. It will be
replaced in the generated parser by the value of the `location_formatting`
argument of the parser generator, which defaults to::

  "lineno=start_lineno, col_offset=start_col_offset, "
  "end_lineno=end_lineno, end_col_offset=end_col_offset"

The default is suitable to generate Python AST nodes.


### Variables in the Grammar

A subexpression can be named by preceding it with an identifier and an `=` sign.
The name can then be used in the action, like this:
```
rule_name[return_type]: '(' a=some_other_rule ')' { a }
```

### Syntax error related rules

Rules meant to provide better error reporting on syntax error are useful but
can be tricky:

- they will prevent the parser from back tracking which may not always be desirable.
  In those cases one can customize the parser to delay the error reporting.
- secondly when used in the alternative of another rule, this alternative will never
  evaluate its action. This can be annoying to measure the code coverage on the parser.
  To alleviate this issue, all rule alternatives making use of a rule whose name
  starts with `'invalid'` will have its action set to `UNREACHABLE` if no action was
  specified. `UNREACHABLE` is a special action  which will be replaced by the value
  of the `unreachable_formatting` which defaults to `None  # pragma: no cover`.

.. note::

  Rules making use of the `&&` forced operator to generate syntax error will never
  run their action and need to be manually annotated.


### Customizing the generated parser

By default, the generated parser inherits from the Parser class defined in pegen/parser.py,
and is named GeneratedParser. One can customize the generated module by
modifying the header and trailer of the module generated by pegen. To do so one
can add dedicated sections to the grammar, which are discussed below:

@class NAME

  This allows to specify the name of the generated parser.

@header

  Specify the header of the module as a string (one can typically use triple
  quoted string). This defaults to MODULE_PREFIX by default which is defined in
  pegen.python_generator. In general you should not modify the header since it
  defines necessary imports. If you need to add extra imports use the next
  section. Note that the header is formatted using `.format(filename=filename)`
  allowing you to embed the grammar filename in the header.

@subheader

  Specify a subheader for the module as a string (one can typically use
  triple quoted string). This is empty by default and is the safer to edit to
  perform custom imports.

@trailer

  Specify a trailer for the module which is appended to the parser definition.
  It defaults to MODULE_SUFFIX which is defined in pegen.python_generator.
  Note that the trailer is formatted using `.format(class_name=cls_name)`
  allowing you to reference the created parser in the trailer.


The following snippets illustrates naming the parser MyParser and making the
parser inherit from a custom base class.

```
@class MyParser

@subheader '''
from my_package import BaseParser as Parser
'''

```

Style
-----

This is not a hard limit, but lines longer than 110 characters should almost
always be wrapped.  Most lines should be wrapped after the opening action
curly brace, like:

```
really_long_rule[expr_ty]: some_arbitrary_rule {
    _This_is_the_action }
```
