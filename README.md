# How to generate an AST
## Motivation
After reading thousands of online articles i could not wrap my head around AST. After a millenia I found a way to do that. So, I decided to write an article about how to do that.
## Programming Language for coding
We will be using the famous (or not very famous) cpp Language to discuss code generation for AST. This article presumes that you know OOP Paradigm in cpp.
## Let's start
Abstract Syntax Tree or AST (for short) is a tree that captures each and every construct of a language in a hierarchical fashion. They are mosty used to generate Intermediate Representation of a Programming language. 
> Intermediate Representation somewhat resembles Assembly Language but it is easily understandable. e.g Three Address code or TAC (for short) and [LLVM-IR](https://llvm.org/docs/LangRef.html) etc.

Before coding let's have a basic understanding of AST so we can all be on the same page.

<p align="center">
  <img src="https://i.stack.imgur.com/dhd3v.png" >
</p>

There are many nodes in this picture. Let's consider the left most node which corresponds to assignment operator. The Verbal description of this node can be "assign a value of 1 to variable x". Great huh? That's where things are about to get interesting. After a quick count we can see that there are 12 nodes in this picture. What if i say that there are actually only 4 nodes from coding perspective. Yes, that's what made me lose my mind at first. The 4 nodes are:

<p align="center">
  <img src='https://www.linkpicture.com/q/dhd3v.png'></a>
</p>

Yes! the representation is totally different from coding perspective and I don't know why :(

Let's see the classes for these nodes.
The class for assignment operator will look like this
```cpp
class AssignmentExpAST {
  string var; //variable to be assigned a value
  int value; //value to be assigned
};
```
You can have your own way of representig identifiers and value.
Now we know what was that "x" and "1" in left "=" node. They were actually data members, not childrens of "=" node.
Before moving on to the expression part (right side on above picture), let's discuss what is that empty node on the top?
Any guesses? May be try to think it on your own and then scroll down for the answer?

So, as many of you would have guessed correctly it was the parent node (it is parent to every other node in our AST).
The class of this node can be as follows:
```cpp
class ExprAST {
};
```
Empty class? yes an Empty class just to make a parent of all other nodes. Simple right? In some cases we make this pure virtual (I will explain it in some future article) which is beyond the scope of this article.

The AssignmentExpAST class becomes the child of ExpAST
```cpp
class AssignmentExpAST :public ExprAST{
  string var; //variable to be assigned a value
  int value; //value to be assigned
};
```
Note: Sometimes we don't have to generate AST for assignment, as it just adds a value in [symbol table](https://www.geeksforgeeks.org/symbol-table-compiler/). But, we will be making an AST for assignment statement as well.<br>
Let's move on to the Expression part which is a little tricky<br>
Expression basically are of two types
1) Binary e.g., 4+2, 5*9 
2) Singular e.g., 4, 5

Binary Expression is made up of singulars and an expression can be made up of many binary expressions. The expression used in the picture above (right most side), is the AST for expression 3*(x+y). As you can guess that x and y are also singulars. They are [Identifiers](https://docs.microsoft.com/en-us/cpp/c-language/c-identifiers?view=msvc-170#:~:text=%22Identifiers%22%20or%20%22symbols%22,are%20reserved%20for%20special%20use.) that holds a value in them.
You guys might be wondering how the computer (or a compiler) knows to evaluate x+y first and then multiply the result with 3.
There is something called a [Translation Scheme](https://www.csd.uwo.ca/~mmorenom/CS447/Lectures/Translation.html/node3.html#:~:text=A%20TRANSLATION%20SCHEME%20is%20a,action%20is%20to%20be%20executed) in which we set [Operator Precedence](https://en.cppreference.com/w/cpp/language/operator_precedence) (which operator to evaluate first?) to tell the compiler which part to evaluate first.
This translation scheme is also responsible for generating this AST (a complete example is provided in this repository).<br>
The code for BinaryExprAST class will look like
```cpp
class BinaryExprAST :public ExprAST{
  char OP;
  ExprAST* left;
  ExprAST* right;
};
```
It includes the OP to be evaluated, the left part and the right part. Notice that the left and right part are of type ExprAST* meaning they are pointers to ExprAST (which is the parent to every construct in our language). Why do we do that? Well, in order to answer this lets go back to the picture above. 
<p align="center">
  <img src='https://www.linkpicture.com/q/dhd3v-1_1.png'></a>
</p>

As we can see that the left or right part can also be another expression that's why we made left and right of BinaryExprAST the pointers to ExprAST. Although the classes types are different we can map every construct to its parent node (cpp OOP concepts) so, whenever we make a pointer to a child node we will be casting it to ExpAST (parent to all node). This simple explanation is the core of AST generation. NumberNode and IdentifierNode classes are missing let's make those classes
```cpp
/*these nodes are terminals so no children*/
class NumberExpNode :public ExprAST{
  int num;
};
class IdentifierExpNode :public ExprAST{
  string identifier;
};
```
This concludes the definition of classes for AST. Phew!!!

## AST Generation
In order to generate AST there is something called a translation scheme. Translation scheme is the backbone of any compiler and is fairly easy to make and code. There are many things to be kept in mind while writing a translation scheme. Without further ado let's make a grammar for our compiler that supports assignments and expressions.
```cpp
start -> statement start 
start -> ^ 
statement -> assignment 
statement -> expression
assignment -> IDENTIFIER ASSIGN INTEGERVAL
expression -> term exp_dash
exp_dash -> + term exp_dash
exp_dash -> - term exp_dash 
exp_dash -> ^ 
term ->       factor term_dash 
term_dash -> * factor term_dash 
term_dash -> / factor term_dash 
term_dash -> ^ 
factor -> INTEGERVAL 
factor -> IDENTIFIER
factor -> (expression)
paren -> LBRACE expression RBRACE 
```


NOTE: ^ represents null <br>
Difficult? well this is the most difficlt phase in compiler generation but once you get the hang of it, it's easy. This translation scheme above will make our dream of AST generation come true. The small case words above are all rules but the capital case words are tokens. Compiler do Lexical Analysis on source code to make Tokens and then parser uses those tokens to do syntax and sementic Analysis. Refer to [Phases of Compiler](https://www.guru99.com/compiler-design-phases-of-compiler.html) for more details.<br>

let's add sementic rules in above grammar to make it a translation scheme.<br>
```cpp
start -> statement start 
start -> ^ 
statement -> assignment 
statement -> expression
assignment -> IDENTIFIER ASSIGN INTEGERVAL {assignment.s = new AssignmentExpAST(IDENTIFIER.lexeme, atoi(INTEGERVAL.lexeme));}

expression -> term  {exp_dash.i = term.s;}
              exp_dash {expression.s = exp_dash.s;}
exp_dash -> + term {exp_dash.i = new BinaryExprAST("+", term.s, exp_dash.i);}
              exp_dash {exp_dash.s = exp_dash.s;}
exp_dash -> - term {exp_dash.i = new BinaryExprAST("-", term.s, exp_dash.i);}
              exp_dash {exp_dash.s = exp_dash.s;}
exp_dash -> ^ {exp_dash.s = exp_dash.i;}
term ->       factor {term_dash.i = factor.s;}
              term_dash {term_dash.s = term_dash.s;}
term_dash -> * factor {term_dash.i = new BinaryExprAST("*", factor.s, term_dash.i);}
               term_dash {term_dash.s = term_dash.s;}
term_dash -> / factor {term_dash.i = new BinaryExprAST("/", factor.s, term_dash.i);}
               term_dash {term_dash.s = term_dash.s;}
term_dash -> ^ {term_dash.s = term_dash.i;}

factor -> INTEGERVAL {factor.s = new NumberExpNode(atoi(INTEGERVAL.lexeme));}
factor -> IDENTIFIER {factor.s = new IdentifierExpNode(IDENTIFIER.lexeme);}
factor -> (expression) {factor.s = expression.s;}
paren -> LBRACE expression RBRACE {paren.s = exp.s}
```
Note: ".s" means that you have to return this value from a function and ".i" means it is bieng passed down as a function parameter. Don't Worry about it too much at this point Coding will make it crystal clear. Was it easy? No? But, it cannot be more easier than this. With this our AST has been generated. WHOA! 🙂

All that's left is to code above translation scheme so let's do this.
```cpp
/*Parsing starts from here*/
/*The backbone of a compiler this generates AST*/
/*function forward declaration*/
bool statement();
ExprAST* expression();
ExprAST* assignment();
ExprAST* factor();
ExprAST* term();
ExprAST* term_dash(ExprAST*);
ExprAST* exp_dash(ExprAST*);
vector<token> tokens;//to hold tokens generated by Lexiacl Analyzer
int i = 0;//index to keep track of next token
token getNextToken()
{
	return tokens[i++];
}
token getToken()
{
	return tokens[i];
}
token expect(string type)
{
	token t = getNextToken();
	if(t.type != type)
	{
		cout << "syntax error" << endl;
		exit(0);
	}
	return t;
}
//start->statement start
//start ->^
void start()
{
	if(statement())
	{
		start();
	}
}
//statement->assignment
//statement->expression
bool statement()
{
	if (assignment())
		return true;
	else if (expression())
		return true;
	return false;
}
//assignment -> IDENTIFIER ASSIGN INTEGERVAL {assignment.s = new AssignmentExpAST(IDENTIFIER.lexeme, atoi(INTEGERVAL.lexeme));}
ExprAST* assignment()
{
	if (getToken().type == "ID")
	{
		token t1=expect("ID");
		expect("ASSIGN");
		token t2=expect("INTEGERVAL");
		return new AssignmentExpAST(t1.lexeme, atoi(t2.lexeme.c_str()));
	}
	return NULL;
}
//expression->term{ exp_dash.i = term.s; }
//exp_dash{ expression.s = exp_dash.s; }
ExprAST* expression()
{
	ExprAST* left = term();
	if (left)
	{
		return exp_dash(left);
	}
	return NULL;
}
//exp_dash -> + term{ exp_dash.i = new BinaryExprAST("+", term.s, exp_dash.i); }
//				exp_dash{ exp_dash.s = exp_dash.s; }
//exp_dash -> - term{ exp_dash.i = new BinaryExprAST("-", term.s, exp_dash.i); }
//				exp_dash{ exp_dash.s = exp_dash.s; }
//exp_dash ->^ {exp_dash.s = exp_dash.i; }
ExprAST* exp_dash (ExprAST* exp)
{
	if(getToken().type=="PLUS")
	{
		expect("PLUS");
		ExprAST* val = term();
		return exp_dash(new BinaryExprAST('+', exp, val));
	}
	else if(getToken().type == "MINUS")
	{
		expect("MINUS");
		ExprAST* val = term();
		return exp_dash(new BinaryExprAST('-', exp, val));
	}
	else
	{
		return exp;
	}
}
//term->factor{ term_dash.i = factor.s; }
//		term_dash{ term_dash.s = term_dash.s; }
ExprAST* term()
{
	ExprAST* exp = factor();
	return term_dash(exp);
}
//term_dash ->* factor{ term_dash.i = new BinaryExprAST("*", factor.s, term_dash.i); }
//				term_dash{ term_dash.s = term_dash.s; }
//term_dash ->/ factor{ term_dash.i = new BinaryExprAST("/", factor.s, term_dash.i); }
//			    term_dash{ term_dash.s = term_dash.s; }
//term_dash ->^ {term_dash.s = term_dash.i; }
ExprAST* term_dash(ExprAST* exp)
{
	if (getToken().type == "MUL")
	{
		expect("MUL");
		ExprAST* val = factor();
		return exp_dash(new BinaryExprAST('*', exp, val));
	}
	else if (getToken().type == "DIV")
	{
		expect("DIV");
		ExprAST* val = factor();
		return exp_dash(new BinaryExprAST('/', exp, val));
	}
	else
	{
		return exp;
	}
}

//factor->INTEGERVAL{ factor.s = new NumberExpNode(atoi(INTEGERVAL.lexeme)); }
//factor->IDENTIFIER{ factor.s = new IdentifierExpNode(IDENTIFIER.lexeme); }
//factor -> (expression) { factor.s = expression.s; }
ExprAST* factor()
{
	if (getToken().type == "INTEGERVAL")
	{
		token t1 = expect("INTEGERVAL");
		return new NumberExpNode(atoi(t1.lexeme.c_str()));
	}
	else if (getToken().type == "ID")
	{
		token t1 = expect("ID");
		return  new IdentifierExpNode(t1.lexeme);
	}
	else if (getToken().type == "LBRACE")
	{
		expect("LBRACE");
		ExprAST* exp = expression();
		expect("RBRACE");
		return exp;
	}
	return NULL;
}
```
Phew 😮‍💨 As you can see that there is a function against every rule and coding is done keeping the translation scheme in view. <br>
How did i code that? Don't worry it's called [recursive descent parsing](https://www.geeksforgeeks.org/recursive-descent-parser/). With this your AST has been generated. You can play around with the code and do some experiments.
> Complete code is present in this repository named "AST.cpp"

You can also contact me at ehteshamarif77@gmail.com, I would be happy to help you the best I could.
