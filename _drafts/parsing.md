Parsing text can be a difficult task to do, specially when this text is written by a human, the most error-prone element in a computer system.

A parser is a program that takes some kind of structured or semi-structured text and spits out some (equivalent) representation of the text that is eaisier to process.

In a programming language, the representation could be an abstract syntax tree (just a fancy name for a normal tree which represents statements in a program) that can be executed directly or processed for other analysis like static typing or transformed to another representation. Another representation could be a bytecode (think high level machine code) which is executed by a virtual machine (generally a loop that takes bytecode and execute it instruction by instruction).

In other applications, the representation could be anything depending on what you need to extract from the text but an AST is a very popular choice. A simple representation is simply storing the parts of text in a dictionary, that's specially true for a regular language.

In this blog post I'm hoping not just to show you how to write a parser, but how you could have arrived at it by yourself. As it turns out, writing a parser is not as tough as one may think, and if you are like me you will find parsers a pleasant to write (to some extent).

# The Lexer

A step that is generally performed before parsing is lexing.

A lexer takes a sequence of characters and spits words. For example a lexer for mathmetical expressions would take the string '369*15 + 8%5' and spits a list of words like ['369', '*', '15', '+', '8', '%', '5'] or more commonly a data type that holds the characters of the word (called it's lexeme), and the type of the word (number, operator, variable) in which case you have got a token (The pair of a word and it's type). Even more often additional data is added to a token like the word's position in the text, it's length, and line.

Notice how even if the lexemes aren't seperated with a space the lexer knows where a lexeme ends. That's because tokens have certain rules. For example in simple mathematical expressions, a token can only be a number or an operator, and it cannot consist of a combination of an operations and digits. Note that a different language may choose to represent the whole mathematical expression as a single token like a string in a programming language, or several tokens get the same token type. Literals for example can be numbers, strings, or any constant the language supports such as `true` and `false`.

As such we need to define some rules as what forms a token and what are the type of each token, but I'm hoping to implement more than one parser in this post. 
===============================================================================

# Problem definition

First things first, we need to specify what we are parsing. It needs to be structured text. Text that follows certain rules. These rules are a language's grammar.

We will start with something simple. Let's say we want to parse a simple english sentence that follows the structure subject-verb-object. Here's a few valid sentences:

- He plays soccer
- She found me

And a few invalid statements:

- He rocks -> missing object
- I -> missing verb and object

I won't specify a formal grammar for this language yet, instead we will dive immediatly into implementation

A lexer for this language is very simple since there's only one token type:
```c
typedef struct sToken {
   // we will just point to the first char in the sentence
   char *lexeme;
   // we need this to know where the token's lexeme ends
   int length;
   // we will generate a linked list of tokens when lexing
   struct sToken *next; 
} Token;

Token * lex(char *text) {
   char *current = text;
   Token *head = malloc(sizeof(Token));
   Token *last = head;
   while (*current != '\0') {
      char *start = current;
      while (*current != ' ' && *current != '\0') {
         ++current;
      }
      // create a new token
      Token *token = malloc(sizeof(Token));
      token->lexeme = start;
      token->length = current - start; // pointers arithmetic
      // build the linked list
      last->next = token;
      last = token;
      // skip space characters.
      while (*current == ' ') {
         ++current;
      }
   }
   return head;
}
```
This is because we are using english words which are seperated by a space character.

For error reporting we may want to store each word's position, and line. If the language supports more types of words, we should also recognize the type of the token in the lexer.

with this, we can start writing a simple parser. The representation I want out of the text is a simple list of the different parts of the sentence.
Before jumping into the parsing code, I would like to introduce a few helper:
```python
class Parser:
   def __init__(self, tokens):
      self._tokens = tokens
      self._current_pos = 0
   
   def advance(self):
      if self.is_at_end():
         return '\0'
      self._current_pos += 1
      return self._tokens[self._current_pos - 1]

   def is_at_end(self):
      return self._current_pos >= len(self._tokens)
```

```python
class Parser:
   def __init__(self, tokens):
      self._tokens = tokens
      self.current_token = 0
   
   def parse(self):
      sentence = []
      sentence.append(self.subject())
      sentence.append(self.verb())
      sentence.append(self.object())
      return sentence
   
   def subject(self):
      if self.is_at_end():
         raise Error("Expected a subject")
      token = self.advance()
      return Subject(token)

   def verb(self):
      ...
   
   def object(self):
      ...
```

I have neglected some of the code in this snippet, but we will get back to it.
Now back to our examples. Here is more invalid sentences:
- Mountain I climbed -> Wrong order (I know 'the' or 'a' is missing. We will get to them soon)
- We you me -> you is not a verb

but our current parser don't care about the word itself, that is it can't differentiate a verb from a subject. To fix this we need to tell our parser what represents a subject, a verb and an object.
The way we do this is pretty simple. We provide it with a list of valid subjects/verbs/objects and test if the parsed subject/verb/object belongs to it. But a better implementation would recognize the type of the word (subject, verb, helping verb, object, etc) in the lexer because the lexer already walked the word's characters so it may as well recognize the word's type.
I won't show how a lexer may do this, but it's very straight forward. From now on I will assume that any token has the following data:
```python
token.type # 'SUBJECT', 'VERB', 'OBJECT', etc or constants (enum for example)
token.lexeme # the token itself
token.line # the line in which the lexeme is on.
token.start # the position of the lexeme in the text from the beginning of its line
```

```python
   def subject(self):
      ...
      
      if token.type != 'SUBJECT':
         raise Error("Error at {}:{}\n\t{} is not a valid subject".format(
            token.line, token.start, token.lexeme)
         )

      return Subject(token)
```

Now we talking! But wait, this parser only parses very simple and limited sentences. What if we have sentences like:
- he bought a car
- the man achieved his dream

To fix this we need to introduce optional tokens in subject and object.
```python
   def subject(self):
      ...
      subject_tokens = []
      if token.lexeme in ('a' or 'the'):
         subject_tokens.append(token)
         token = self._tokens[self.current_token]
         self.current_token += 1
      
      ...
      subject_tokens.append(token)
      return Subject(*subject_tokens)
```

Note: There's many ways to do this but for now I will go with the simpler approach

Now our subjects and objects are not limited to one word, but the poor verbs are still limited. What is worse is that we can't use what we used for the subject and object because verbs are more complicated:
- He is playing soccer -> valid
- He is plays socces -> invalid

certain verbs are accepted with 'is', others aren't. To do this we will split the verb into two types.
```python
   def verb(self):
      if self._tokens[self.current_token] == 'is':
         return self.is_verb()
      return self.normal_verb()
   
   def is_verb(self):
      is_token = self._tokens[self.current_token]
      self.current_token += 1
      verb = self._tokens[self.current_token]
      if verb not in accepted_verbs_with_is:
         raise Error("{} is not a valid verb after 'is'".format(verb))

      self.current_token += 1
      return Verb(helping_verb=is_token, verb=verb)
   
   def normal_verb(self):
      verb = self._tokens[self.current_token]
      self.current_token += 1
      if verb not in accepted_noraml_verbs:
         raise Error("{} is not a valid verb".format(verb))
      return Verb(verb=verb)
```

And that's it for our first parser. It parses simple english sentences and validates it's correctness.
All that remains is the Subject, Verb, and Object classes, but these are simple structures for holding data. So I won't bother showing their code. We could have simply returned the lists but it's nicer to have classes.

Explaining this till now without using any terminology have been a real pain, so a little break from the code and a brief introduction to some terms.

A parser digests a `language`. A language is the set of all valid sentences (combination of words).
The `grammar` of a language is the set of rules that specify whether a certain sentence is part of the language or not.
We didn't write a grammar for our previous language, but the code pretty much describes it. We can write the grammar of a language in many ways. Some grammars can be descibed visually like a finite state machine, but the most powerful way to describe a language, somewhat suprisingly, is another language. The BNF (Buck-Naur Form) is a grammar specification language, which nicly can be deduced from our previous code. Here is our language's grammar in BNF:
```
sentence := subject verb object

subject := ('a'|'the')? valid_subjects

verb := 'is' is_verb | simple_verb
is_verb := valid_verbs_with_is
simple_verb := valid_normal_verbs

object := ('a'|'the')? valid_objects

valid_subjects := 'man' | 'she' | ...
valid_verbs_with_is := 'playing' | 'running' | ...
...
```
the `|` symbol means or, and the '?' symbol means optional.
From now on we will use this form to describe a language before diving into code.