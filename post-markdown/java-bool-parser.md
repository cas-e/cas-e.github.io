Generating Truth Tables in Java : Part 2 : Parsing
===

Part one of this series can be found [here](java-bool-expr.html).

The full code developed in this post can be found [here](https://github.com/cas-e/blog-code/tree/main/JavaBoolParse).

The plan
---

We wrapped up last time with a working boolean expression evaluator and a to-do list for our problem that looked something like this:

~~~
generate all truth tables given a formula
├── make sense of the input text
│   ├── tokenizing ✅
│   └── precedence rules
├── find every possible value for the variables 
├── evaluate all possible expressions from the formula
│   └── evaluate a single expression ✅
└── print the results in the form a table
~~~

Today we tackle precedence rules. I see two options here:

* Write a parser, which will convert the precedence syntax into a tree
* Just don't have precedence rules -- use pre(or post)-fix syntax instead

Last time we chose to just work with prefix syntax. We could keep using that strategy and finish the rest of the project. No parser needed. Just delete it from the to-do list and write it to the unfulfilled-wish list. That would be fine, as long as we are comfortable writing our boolean expressions in prefix form. And there is good reason to get comfortable with it if we aren't already-- parsing is actually the trickiest part of this entire problem. It's also the most inessential. Writing parsers is perhaps best avoided when possible. 

But let's do it anyways just for fun. 

A little recap
---

Last time, we were able to take our tokens and feed them directly into the parser. The parser only needed to press the `take` button over and over to be able to get to traverse the expression tree, so it could eventually find values and start reducing the expression:
 
~~~
public static boolean eval(Seq ops) {
    String op = ops.take();
    if (op.equals("true"))    return true;
    if (op.equals("false"))   return false;
    if (op.equals("not"))     return !eval(ops);
    // etc...
~~~

That strategy won't work anymore with precedence syntax. Take a look:

~~~
true && false && false || true
~~~

Here, the root of the expression tree is the `||` near the end of the expression. And in general with this kind of syntax, we don't really know where to start until we have read the whole thing. We could make `eval` jump around all over the tokens trying to remember where it had been... but that would get complicated. Instead, we write a parser that just transforms precedence syntax into a tree. 

The method we will use to parse is often called Pratt Parsing, though it goes by other names as well. And our implementation of this method is essentially a version of the one in Alex Kladov's [Simple but Powerful Pratt Parsing](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html#Simple-but-Powerful-Pratt-Parsing). Of all articles I looked at for this topic, Alex's was my favorite. It has a clear explanation of the problem, and a direct implementation strategy that easily translates to other languages besides Rust. 

Let's get started.

Organization
---

Here is how we will organize our project now:

~~~
Project/
├── TruthTable.java  // contains main()
└── Tree.java        // provides parse() and defines what a tree is
~~~

Let's start with a look at TruthTable.java. Here is it is in its entirety:

~~~
public class TruthTable {
	public static void main(String[] args) {
		String   input = "false || true  && (!false && true);
		Tree.Node tree = Tree.parse(input);
		boolean answer = eval(tree);

		System.out.println(answer);
	}
	public static boolean eval(Tree.Node n) {
		if (n.val.equals("true"))  return true;
		if (n.val.equals("false")) return false; 
		if (n.val.equals("!"))     return !eval(n.left);
		if (n.val.equals("||"))    return eval(n.left) || eval(n.right);
		if (n.val.equals("&&"))    return eval(n.left) && eval(n.right);
		System.out.println("unrecognized token in evaluator");
		System.exit(1);
		return false;
	}
}
~~~

All it has right now is the `eval` method. We will update it more in future posts. But for now, look how little about it has changed from last time-- mostly just the fact that it now takes `Tree.Node` as input. 

All of our other work in this post will be building out the Tree class in Tree.java. 

Updating the Tokenizer
---

The tokenizer now needs to recognize and separate off the follow strings: `true`, `false`, `&&`, `||`, `!`, and also `(` and `)`...

~~~
public static String[] tokenize(String input) {
    String s = input.replace("(", " ( ").replace(")", " ) ").replace("!", " ! ");
    String[] sa = s.replace("&&", " && ").replace("||", " || ").split("\\s+");
    return Arrays.stream(sa).filter(x -> x.length() > 0).toArray(String[]::new);
}
~~~

This is an inefficient way to produce tokens, since every call to replace is creating a new underlying char array. And the more tokens we add, the worse it gets. **AND** it can't even handle newlines or other whitespace! All our inputs better be one-liners only. Which kind of solves the inefficiency problem since our input size is tightly bounded...

Anyways, it's fine for our purposes right now. And we could build it out later without changing our other code if we keep the type signature the same.

Representing Trees
---

Let's take a look at `Tree.Node`:

~~~
public static class Node {
	String val;
	Node left;
	Node right;
	boolean isVar = false;
	public Node(String s, Node l, Node r) {
		val = s;
		left = l;
		right = r;
	}
}
~~~

Notice that the left and right fields contain pointers to other Node instances.
This is what allows us to build trees. The convention will be that leaves like (`true` and `false`) will have `null` in their left and right branches. Unary functions, of which we have only one named `!`, will contain null in the right branch only.

For example, the expression:

~~~
true && !false
~~~

Would have its tree be represented by:

~~~
Node trueLeaf = new Node("true", null, null);
Node falseLeaf = new Node("false", null, null);
Node notSingle = new Node("!", falseLeaf, null);
Node tree =  Node("&&", trueLeaf, notSingle);
~~~


Writing the Parser: Preliminaries
---

Here is our `parse` method that we export to our main TruthTable class:

~~~
public static Node parse(String input) {
	return parseExpr(tokenize(input));
}
~~~

It calls `tokenize`, which we already defined, and `parseExpr` which looks like this:

~~~
public static Node parseExpr(String[] tokens) {
	Seq toks = new Seq(tokens);
	return parseInfix(toks, 0);
}
~~~

We will need to add some more functionality to our Seq class:

~~~
private static class Seq {
	int index;
	String[] array;
	public Seq(String[] s) { array = s;}
	public boolean more() { return (array.length - index) > 0; }
	public String take() { return array[index++]; }
	public String peek() { 
		if (!more()) {
			return "";
		}
		return array[index];
	}
~~~

The `take` method is there as before, but we have added a `more` predicate to determine the end of the input. `peek` is like `take`, but it doesn't move the input forward, and it also signals the end of input as the empty string. 

Now, about that `0` that we passed into `parseInfix` earlier, that all has to do with how we represent precedence. We provide a lookup to see the precedence rules from a given operator here:

~~~
private static int[] bindingPower(String s) {
	if (s.equals("||")) { return new int[]{1, 2};}
	if (s.equals("&&")) { return new int[]{3, 4};} 
	return new int[]{-1, 5}; // case for !
}
~~~

The returned arrays are representing pairs of values: the left and right binding power of the operators. For the case of `!`, the left side `-1` signals that it has no left binding power at all. Here, ops with higher numbers are said to "bind tighter" and end up closer to the leaves of the tree. The left binding powers all have looser binding, which encodes the fact that ties break to the left. 

Writing the Parser: Small Version
---

First we write a parser for just `true`, `false`, `&&`, and `||`. Later versions will add the rest and some error handling, but this is the basic idea:

~~~
private static Node parseInfix(Seq toks, int minbp) { 
	Node lhs = parsePrefix(toks);

	while (toks.more()) {
		String op = toks.peek();

		int lbp = bindingPower(op)[0];  

		if (lbp < minbp) { 
			break;
		}

		toks.take();
		Node rhs = parseInfix(toks, rbp);  // recursive call within loop (!)
		lhs = new Node(op, lhs, rhs); 
	}
	return lhs;  // notice if the loop fails to fire,
				 // left-hand side is the only side
}				 // this is also how we handle immediates

private static Node parsePrefix(Seq toks) {
	String tok = toks.take();
	if (tok.equals("true") || tok.equals("false")) {
		return new Node(tok, null, null);
	}
	return null; // unrecognized
}
~~~

Don't worry if you can't understand it at first, it definitely took me a minute to get this. One thing I sometimes do if I am stuck on understanding something is to "become the Turing machine". By which I mean, I try to write down on paper which values are changing, step by step, for some small example  inputs. Eventually things will click. And again, I highly recommend the explanation [here](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html#Simple-but-Powerful-Pratt-Parsing). The code here is pretty much a direct translation, minus some of the Rust-specific type stuff. 


Writing the Parser: Full Version
---

Let's handle some errors. Even though we are only parsing a single line, there are many ways for an expression to be malformed and not valid for evaluation. So we'll add a helper:

~~~
private static void error(String s) {
    System.out.println(s);
    System.exit(1);
}
~~~

And here is our updated `parseInfix` 

~~~
private static Node parseInfix(Seq toks, int minbp) {
	Node lhs = parsePrefix(toks);
	while (toks.more()) {
		String op = toks.peek();
		if (op.equals(")")) {
			break;
		} 
		if (!(op.equals("&&") || op.equals("||"))) {
			error("bad operator");
		}
		int lbp = bindingPower(op)[0];
		int rbp = bindingPower(op)[1];
		if (lbp < minbp) {
			break;
		}
		toks.take();
		Node rhs = parseInfix(toks, rbp);
		lhs = new Node(op, lhs, rhs);
	}
	return lhs;
}
~~~

It now has an error condition to make sure only our infix operators are used in `parseInfix`. This filters out programs like `false true false`, where applying `true` to these `false` values makes no sense. We also add some logic to break the loop for closing parens. 

Here is our updated parsePrefix:

~~~
private static Node parsePrefix(Seq toks) {
	if (!toks.more()) {
		error("unexpected end of line");
	}
	String tok = toks.take();
	if (tok.equals("true") || tok.equals("false")) {
		return new Node(tok, null, null);
	}
	if (tok.equals("!")) {
		int rbp = bindingPower(tok)[1];
		return new Node("!", parseInfix(toks, rbp), null);
	}
	if (tok.equals("(")) {
		Node node = parseInfix(toks, 0);
		if (!toks.peek().equals(")")) {
			error("unbalanced paren");
		}
		toks.take();
		return node;
	}
	if (tok.equals("&&") || tok.equals("||") || tok.equals(")")) {
		error("bad operand");
	}
	Node node = new Node(tok, null, null);
	node.isVar = true;
	return node;
}
~~~

We've added error checking here as well, tossing away many syntactically invalid expressions like `"|| ( &"` or `"& false true"`. Our definition also rejects the empty expression represented by either the empty string or just spaces. The last error we check is to make sure none of our binary operators are in the prefix position. 

All of that case handling and error checking produces a sieve that ensures that, at the end of this function, anything else is a variable. We aren't doing anything with variables in this post but we just set this up now for next time. 

As far as added functionality in parsePrefix goes, notice how the case for `(` calls `parseInfix` with the lowest minimum binding power, since parens delineate a new sub expression. Meanwhile, the case for `!` calls with the highest possible binding power for this language.


Wrap up
---
We can now go back into our Truth Table class and use our `parse` method from the Tree class:

~~~
public class TruthTable {
	public static void main(String[] args) {
		String   input = "false || true  && (!false && true)";
		Tree.Node tree = Tree.parse(input);
		boolean answer = eval(tree);

		System.out.println(answer);
	}
	// eval...
~~~

Which prints what it should:

~~~
true
~~~

One more item is crossed off our list!

~~~
generate all truth tables given a formula
├── make sense of the input text ✅
│   ├── tokenizing ✅
│   └── precedence rules ✅
├── find every possible value for the variables 
├── evaluate all possible expressions from the formula
│   └── evaluate a single expression ✅
└── print the results in the form a table
~~~


Next time we'll begin work on producing values for our variables.



