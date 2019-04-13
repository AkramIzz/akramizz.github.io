Parsing text can be a difficult task to do, specially when this text is written by a human, the most error-prone element in a computer system.

A parser is a program that takes some kind of structured or semi-structured text and spits out some equivalent representation of the text that is eaisier to process. The text is said to be written in a language or a format.

In a programming language, the representation could be an abstract syntax tree (just a fancy name for a normal tree which represents statements in a program) that can be executed directly or processed for other analysis like static typing or transformed to another representation. Another representation could be a bytecode which is executed by a virtual machine (generally speaking a loop that takes bytecode and execute it instruction by instruction). You can even choose to evaluate the statements directly while parsing without transforming them into another representation.

In other applications, the representation could be anything depending on what you need to extract from the text but an AST (Abstract Syntax Tree) is also a popular choice. A simple representation is simply storing the parts of text in a dictionary, that's specially true for a regular language.

In this blog post I'm hoping not just to show you how to write a parser, but how you could have arrived at it by yourself. As it turns out, writing a parser is not as tough as one may think, and if you are like me you will find parsers a pleasant to write. Well, at least to some extent.

# Problem definition

First things first, we need to specify what we are parsing. It needs to be structured text, text that follows certain rules. These rules are a language's grammar.

We will start with something simple. Mathematical expressions are pretty easy to do, we will support integer numbers, and four operators for now, but we will add another one later:

- addition and subtraction
- multiplication and division

I won't specify a formal grammar for this language yet, instead we will dive immediatly into implementation. And we won't be implementing a parser yet. Instead we will implement a simple function that takes the expression and return the result of the calculation.

# A Calculator, Barely

This should be very simple, we will calculate the result of a mathematical expression with no respect for operations order. It will just calculate from left to right, with all operators having the same precedence, so `45+5*2` will be calculated as `(45+5)*2` resulting in `100`. So we have a calculator, sort of. Let's just dive into code, it's quite straight forward:

```java
float eval(char *expr) {
   char *current = expr;
   float result = number(&current);
   while(current != '/0') { // end of string
      char op = operator(&current);
      float rv = number(&current);
      result = perform_operation(op, result, rv);
   }
   return result;
}

float number(char **current) {
   char *start = *current;
   while(**current >= '0' && **current <= '9') {
      ++(*current);
   }
   return str_to_float(start, *current);
}

char operator(char **start) {
   char op = **start;
   ++(*start);
   return op;
}

float perform_operation(char operator, float lv, float rv) {
   switch(operator) {
      case '+':
         return lv+rv;
      case '-': 
         return lv-rv;
      case '*': 
         return lv*rv;
      case '/':
         return lv/rv; 
   }
}
```

Next let's write this in a recursive way

```java
float eval(float result, char *expr) {
   if (*expr == '\0') { // base case
      return result;
   }
   char op = operator(&expr);
   float rv = number(&expr);
   result = perform_operation(op, result, rv);
   return eval(result, expr);
}
```

I hear you asking why. Well it's shorter for a starter, and it's easier to write a right to left evaluation of the mathematical expression using recursion than using a while loop. For example, we want this `125+25*2-2+8` to be evaluated as `(125+(25*(2-(2+8))))`.

```java
float eval(char *expr) {
   float lv = number(&expr);
   if (*expr == '\0') {
      return lv;
   }
   char op = operator(&expr);
   float rv = eval(expr);
   return perform_operation(op, lv, rv);
}
```

Later on for precedence we will need to switch between left to right and right to left evaluation. `125+25*2` is correctly evaluated as right to left and `300-2+8` is correctly evaluated as left to right.

Although this calculator is very limited in terms of what it can do, it's also limited in terms of what it understands (or can parse correctly). Take this expression `1 + 5`, the above calculator we implemented wouldn't give the right result, 
===
   if it didn't crash.
===
So let's get this issue out of the way.

# The Lexer

To start simple, first thing we need to get out of the way when parsing text is to differentiate the words of text. Take C for example, writing `int x=0;` is no different than writing `int x = 0 ;`, but writing `intx=0;` isn't the same.

A step that is generally performed before parsing is lexing.

A lexer takes a sequence of characters and spits 'words' (called lexemes). For example a lexer for mathematical expressions would take the string '369\*15 + 8%5' and spits a list of lexemes like ['369', '\*', '15', '+', '8', '%', '5'] or more commonly a data type that holds the characters of the lexeme, and the type of the lexeme (number, or operator) in which case you have got a token (The pair of a word and it's type). Even more often additional data is added to a token like the lexeme's position in the text, it's length, and line.

Notice how even if the lexemes aren't seperated with a space the lexer knows where a lexeme ends. That's because tokens have certain rules. For example in simple mathematical expressions, a token can only be a number or an operator, and it cannot consist of a combination of operations and digits. `12/8` is not a token. Note that a different language may choose to represent the whole mathematical expression as a single token like a string in a programming language, or several tokens get the same token type.

As such we need to define some rules as what forms a token and what are the type of each token, which is often very straight forward given that you know what kind of language you want to parse.

A lexer for a mathematical language is very simple, all we need to do is to seperate numbers from operators:

```java
typedef struct _Token {
   // we will just point to the first char in the mathematical expression
   // we can also calculate the position of the first char of the lexeme using
   // pointer arithmetic
   char *lexeme;
   // we need this to know where the token's lexeme ends
   int length;
   // we have two types for now but we will add another later
   // For simplicity an integer is used and all operators are given the same
   // type value.
   // we won't be using this field now, but we will need it later.
   int type;
   // we will generate a linked list of tokens when lexing
   // this is simpler than managing a dynamic array
   struct _Token *next;
} Token;

#define TYPE_NUMBER 1
#define TYPE_OPERATOR 2

Token * lex(char *text) {
   char *current = text;
   Token *head = malloc(sizeof(Token));
   Token *last = head;
   while (*current != '\0') {
      // lexeme start
      char *start = current;
      int type = -1
      if (*current >= '0' && *current <= '9') {
         type = TYPE_NUMBER
         while(*current >= '0' && *current <= '9')
            ++current;
      } else if (*current == '+' || *current == '-' 
      || *current == '*' || *current == '/') {
         type = TYPE_OPERATOR
         ++current;
      }
      // create a new token
      Token *token = malloc(sizeof(Token));
      token->lexeme = start;
      token->length = current - start; // pointers arithmetic
      token->next = NULL
      // update the linked list
      last->next = token;
      last = token;
      // skip space characters
      while (*current == ' ') {
         ++current;
      }
   }
   return head;
}
```

Instead of a linked list, you may also choose to lazy evaluate the lexing while parsing. You maintain a state that tells you where the lexing has stopped and each call to `lex` will generate a single token. You call `lex` while parsing whenever you need the next token.

# Another Calculator

This time we will be using the lexer we just built to evaluate the mathematical expression.

To introduce precedence we won't use the `perform_operation` function with it's simple switch statement; we need to switch between left-to-right and right-to-left evaluation.

We need more control on what operations are evaluated and in which association (left-to-right or right-to-left), and as such we will split `perform_operation` into functions and incorperate `eval` into these functions. Each will try to perform the operation and return the result, but if it fails, it passes the operation to the next function. `eval` just becomes the kicker of the evaluation process. Code describes this better:

```java
float eval(Token *current) {
   float lv = number(&current);
   return addition(lv, &current);
}

float addition(float lv, Token **current) {
   if (*current == NULL) return lv;

   if ((*current)->lexeme != '+')
      return subtraction(lv, current);
   // consume the operator
   *current = (*current)->next;
   float rv = number(current);
   return addition(lv+rv, current);
}

float subtraction(float lv, Token **current) {
   if ((*current)->lexeme != '-')
      return multiplication(lv, current);
   
   *current = (*current)->next;
   float rv = number(current);
   return addition(lv-rv, current);
}

float multiplication(float lv, Token **current) {
   if ((*current)->lexeme != '*')
      return division(lv, current);
   
   *current = (*current)->next;
   float rv = number(current);
   return addition(lv*rv, current);
}

float division(float lv, Token **current) {
   // no need to check the operator.
   *current = (*current)->next;
   float rv = number(current);
   return addition(lv/rv, current);
}
```

With left-to-right evaluation in place, we can start adding the right-to-left evaluation, depending on the operation.

First, here is how we implement a right-to-left evaluation in this setup:

```java
float addition(float lv, Token **current) {
   if (*current == NULL) return lv;

   if ((*current)->lexeme != '+')
      return subtraction(lv, current);
   
   // consume operator
   *current = (*current)->next;
   float rv = number(current);
   return lv+addition(rv, current);
}
```

Really, we just changed one line.

For precedence, we will use a combination of these two. When in `multiplication`, we know we shouldn't evaluate addition or subtraction before evaluating the multiplication as in left-to-right evaluation, but also for multiplication and division a left-to-right evaluation is .. right. In contrary, when in `addition` and a multiplication or division operation is next we know we should evaluate multiplication and division first as in right-to-left evaluation. To translate this into code, we will change that last line in `addition` and `subtraction` that calls `addition` to reflect these relationships. And we will change what happens if the token lexeme isn't the expected operator in `division`.

A right-to-left evaluation is performed, then a left-to-right evaluation is performed. Like how we got rid of the switch statement in `perform_operation`, we don't need to really switch between the two evaluation strategies. Instead when one can't be performed it just returns the `lv` value.

```java
float addition(float lv, Token **current) {
   if (*current == NULL) return lv;

   if ((*current)->lexeme != '+')
      return subtraction(lv, current);
   
   // consume operator
   *current = (*current)->next;
   float rv = number(current);
   float result = lv+multiplication(rv, current); // if no multiplication or division is found, rv is returned.
   return addition(result, current); // if no addition is found, result is returned
}

float subtraction(float lv, Token **current) {
   if ((*current)->lexeme != '-')
      return multiplication(lv, current);
   
   *current = (*current)->next;
   float rv = number(current);
   float result = lv-multiplication(rv, current);
   return addition(result, current);
}

float multiplication(float lv, Token **current) {
   if (*current == NULL) return lv;

   if ((*current)->lexeme != '*')
      return division(lv, current);
   
   *current = (*current)->next;
   float rv = number(current);
   return addition(lv*rv, current);
}

float division(float lv, Token **current) {
   if ((*current)->lexeme != '/')
      return lv;
   
   *current = (*current)->next;
   float rv = number(current);
   return addition(lv/rv, current);
}
```

there's no changes in multiplication and division (except for that `return lv`) due to them being the highest precedence operators. It's important though to go from lowest to highest precedence.

Congrats! if you didn't realize it yet, you have just wrote a very little parser. There's a lot more to learn, we barely scratched the surface of parsers, but these are the basics of a particular parsing technique, namely top-down parsers. If you want to learn more about parsers, just follow the links below:

- 
- 
- 
- 

Explaining this till now without using any terminology have been a real pain, so a little break from the code and a brief introduction to some terms.

A parser digests a `language`. A language is the set of all valid sentences (combination of words).
The `grammar` of a language is the set of rules that specify whether a certain sentence is part of the language or not.
We didn't write a grammar for our previous language, but the code pretty much describes it. We can write the grammar of a language in many ways. Some grammars can be descibed visually like a finite state machine, but the most powerful way to describe a language, somewhat suprisingly, is another language. The BNF (Buck-Naur Form) is a grammar specification language, which can be deduced from our previous code. Here is a very simplified english language's grammar in BNF:
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