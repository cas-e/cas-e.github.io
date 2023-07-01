Generating Truth Tables in Java : Part 1 : Expressions
===

This is part one in a N part series. (I am not sure yet just how many posts it will take). Fair warning: I have only just started learning Java, so some of this will likely not be idiomatic Java. But I do want to write down what I have learned so far to help solidify my understanding, and to get more practice writing in general. Even so, I did choose to write this post in a tutorial form. I honestly think I did this only because, having been immersed in so many tutorials the past few weeks, it is now my default mode of thought. But if this tutorial does actually end up helping anyone, that would be very nice!

The full code developed in this post can be found [here](https://github.com/cas-e/blog-code/blob/main/JavaBoolEval/BoolEval.java) on github.


Goal
---
Our goal here is to write a program that, when given a string representing a  boolean formula like 

~~~
x && y
~~~

the program will produce

~~~
x     y     | formula
---------------------
false false | false
false true  | false
true  false | false
true  true  | true
---------------------
~~~

It should also work on longer boolean formulas, such as:

~~~
p || q && !r && q
~~~

Which will produce

~~~
p     q     r     | formula
---------------------------
false false false | false
false false true  | false
false true  false | true
false true  true  | false
true  false false | true
true  false true  | true
true  true  false | true
true  true  true  | true
---------------------------
~~~

We can divide this problem into four parts:

* making sense of the input text
* finding through every possible value for the variables in the input
* determining a value for each of those variables
* printing the results in the form a table

Those are a lot of things to think about all at once. Maybe too many.

So, we will start with an easier sub-problem. Our goal will be, when given a boolean expression without any variables like

~~~
true && true
~~~

the program will produce

~~~
true
~~~

The parts of our problem are now:

* making sense of the input text
* determining a single value for the expression

That seems manageable.

Making Sense of the Input Text
---

Before we actually get into any coding, lets try to understand some aspects of the problem.

Here is an example of a boolean expression in Java's syntax:

~~~
false || true  && (!false && true)
~~~

A common way to visualize an expression like this is as a tree:

~~~
or
├── false
└── and
    ├── true
    └── and
        ├── not
        │   └── false
        └── true
~~~

If it's not obvious how the tree corresponds to the text, then it is worth taking a moment to look over it. (And remember the precedence rules for the java version: `not` is evaluated before `and`, `and` is evaluated before `or`).

One reason it is nice to visualize the expression in the tree form, is because it tells you how to interpret the statement **without knowing any of the precedence rules**. If you look at the top of the tree, and visually follow the lines, every operator points to its operands, and when you get to the bottom, you can start evaluating the simpler statements and build up the answer.

Let's look at one more way to write a boolean expression: prefix syntax. In prefix syntax, the operators like `and` and `not` come **before** the operands like `true` or `false`. In prefix syntax we can write the above tree as:

~~~
or false and true and not false true
~~~

Which, to me, is hard to make sense of. But if we add some parentheses:

~~~
(or false (and true (and (not false) true)))
~~~

Then it is a little easier to read, though it still takes some getting use to. 

What is nice about the prefix syntax is the same thing that is nice about the tree representation-- we do not need to know any precedence rules to understand it. So, following the above ideas, we can choose to write a program that:

* Scans through the text 
* Understands precedence rules
* Evaluates the corresponding expression

Or alternatively, we can write a program that only needs to do two of those:

* Scans through the text
* Evaluates the corresponding expression

Again we have the opportunity to solve a simpler sub-problem. So for now we will only focus on evaluating expressions in a prefix notation.

Scanning Text
---

Okay, some coding now. The first part of our program will take a string like this:

~~~
"(or false (and true (and (not false) true)))"
~~~

And returns all the important words, or **tokens**, as an array...

~~~
["or", "false", "and", "true", "not", "false", "true"]
~~~

Notice that all occurrences of spaces has been removed, but also of parentheses. Since we know the arity of every operator (e.g. the arity of `and` is two because it needs two arguments),the parentheses are unnecessary. However, we will allow them in the text to keep the inputs readable.

Java has built-in functions [replace](https://docs.oracle.com/javase/8/docs/api/java/lang/String.html#replace-java.lang.CharSequence-java.lang.CharSequence-) and [split](https://docs.oracle.com/javase/8/docs/api/java/lang/String.html#split-java.lang.String-) that can help us here:

~~~
public class BoolEval {
	public static void main(String[] args) {
		String testInput = "(or false true)";
		String[] tokens = tokenize(testInput);
		System.out.print("[");
		for (String t : tokens) { 
			System.out.print(t + " , ");
		}	
		System.out.print("]");
	}
	public static String[] tokenize(String input) {
		return input.replace("(", "  ").replace(")", "  ").split("\\s+");
	}
}
// ==> [ "", or , false , true ,]
~~~

We are almost there. tokenize separates all of the tokens, but it also returns some empty strings in the array. On that note, the tokenizer presented here is almost exactly the one presented in Peter Norvig's essay [(How to Write a (Lisp) Interpreter (in Python))](https://norvig.com/lispy.html). (Which (by the way) is an excellent read). But our tokenizer will have to be somewhat different, due to the differences in Python versus Java. In Python, `split` with no arguments returns the string split by spaces, with the resulting list having no occurrences of the empty string. Java's `split` doesn't have this functionality. (Neither does Go's `strings.Split` by the way). But Java **does** provide us with `lambdas` which will get the job done. Here is the updated method with a `lambda` that filters out all of the empty spaces:


~~~
import java.util.Arrays;

public class BoolEval {
	public static void main(String[] args) {
		String testInput = "(or false (and true (and (not false) true)))";
		String[] tokens = tokenize(testInput);
		System.out.print("[");
		for (String t : tokens) { 
			System.out.print(t + " , ");
		}
		System.out.print("]");
	}
	public static String[] tokenize(String input) {
		String[] s = input.replace("(", "  ").replace(")", "  ").split("\\s+");
		return Arrays.stream(s).filter(x -> x.length() > 0).toArray(String[]::new);
	}
}
~~~

Now that we can generate tokens from strings, we just have to traverse and evaluate them.

Traversing Tokens
---

Here we will make a wrapper class around our string array called `Seq`. `Seq` is pronounced "seek" and will represent a sequence we can seek through to get things. 

~~~
import java.util.Arrays;

public class BoolEval {
	// main() ...
	// tokenize() ...
}

class Seq {
	int index;
	String[] array;
	public Seq(String[] sa) { 
		this.array = sa;
	}
	public String take() { 
		return this.array[this.index++]; 
	}
}
~~~

We can think of `Seq` as a little machine with a button called `take()`. Every time we press the `take()` button, the next thing in the sequence pops out of the machine.


Evaluating expressions
---

Let's add another method to our BoolEval class called `evaluate`:

~~~
public class BoolEval {
	// ...
    public static boolean evaluate(String[] formula) {
    	Seq ops = new Seq(formula);
    	// do something...	
    	return false;
    }
}
~~~

Right now, `evaluate`'s only job is to initialize the array into a `Seq` class, so that outside callers of the method don't have to. We make a method `eval` that will do the actual work:

~~~
public static boolean evaluate(String[] formula) {
	Seq ops = new Seq(formula);
	boolean value = eval(ops);	
	return value;
}
public static boolean eval(Seq ops) {
	// do something...
	return false;
}
~~~

So, what should `eval` do? Lets think of some example expressions...

~~~
(and true false)       ==> false
(or false (not false)) ==> true
true                   ==> true
~~~

We can divide these up into two types: **immediate expressions** like `true` we know the answer to right away. We will call everything else a **compound expression**. So we have two little parts to this

* evaluate immediate expressions
* evaluate compound expressions

So let's do the simple thing and work with immediates.

~~~
public static boolean eval(Seq ops) {
	String op = ops.take();
	if (op.equals("true"))    return true;
	if (op.equals("false"))   return false;

	System.out.println("unrecognized token: " + op);
	System.exit(1);
	return false;
}
~~~

We can now test the only two immediate expressions of the language: `true` and `false`. To use our evaluate method, we can flow the input string through the tokenizer, then flow the tokens into the evaluator:

~~~
public class BoolEval {
    public static void main(String[] args) {
        String testInput = "true";
        String[]  tokens = tokenize(testInput);
        boolean    value = evaluate(tokens);
        System.out.println(value);
    }
    // tokenize() ...

	public static boolean evaluate(String[] formula) {
		Seq ops = new Seq(formula);
		boolean value = eval(ops);	
		return value;
	}
	public static boolean eval(Seq ops) {
		String op = ops.take();
		if (op.equals("true"))    return true;
		if (op.equals("false"))   return false;

		System.out.println("unrecognized token: " + op);
		System.exit(1);
		return false;
	}
}
// Seq() ...
~~~

Since the test input is set to `true`, running this gets us...

~~~
true
~~~

It works! But it almost seems silly to write such a simple program. All that effort just to turn a `String true` into a `boolean true`? But writing in small pieces like this affords us the ability to compile, run, and test our programs frequently. Which, for me at least, is crucial. I am frequently making little type and syntax errors as I write, an it is nice to have these caught early by the Java compiler. 

Moving on now. We have immediates handled, but what about compound expressions? Lets take another look at at the example tree representation of expressions from earlier:

~~~
or
├── false
└── and
    ├── true
    └── and
        ├── not
        │   └── false
        └── true
~~~

To evaluate this, we can look at the top element, then look at it's first branch. So, Look at `or`, then look at `false`. We know the value of `false`, it's false. Easy. We are going to compute `or(false, *something*)`. The next branch of `or` is `and`. We don't know the value of that yet. So we do the same thing. Look at `and`, then look at `true`. We know the value of `true`, it's true. Easy. We are going to compute `or(false, and(true, *something*))`. The next branch of `and` is...


And so on. See the repetitive pattern? This pattern allows us to describe `eval` [recursively](https://www.freecodecamp.org/news/how-recursion-works-explained-with-flowcharts-and-a-video-de61f40cb7f9/#:~:text=A%20recursive%20function%20always%20has,the%20function%20stops%20calling%20itself.) as follows:

~~~
public static boolean evaluate(String[] formula) {
	Seq ops = new Seq(formula);
	boolean value = eval(ops);	
	return value;
}
public static boolean eval(Seq ops) {
	String op = ops.take();
	if (op.equals("true"))    return true;
	if (op.equals("false"))   return false;
	if (op.equals("not"))     return !eval(ops);
	if (op.equals("or"))      return eval(ops) || eval(ops);
	if (op.equals("and"))     return eval(ops) && eval(ops);

	System.out.println("unrecognized token: " + op);
	System.exit(1);
	return false;
}
~~~

This recursive method walks down the list of instructions until it finds an immediate value. Java saves that partial answer somewhere, until it gets two partial answers that it can call one of the operations on. (Or just one in the case of `not`). It keeps doing that until the whole thing is reduced and the  goal of finding a single boolean value has been achieved. 


To use our new evaluate method, we flow the input string through the tokenizer again, then flow the tokens into evaluate, just like before:

~~~
import java.util.Arrays;

public class BoolEval {
    public static void main(String[] args) {
        String testInput = "(or false (and true (and (not false) true)))";
        String[]  tokens = tokenize(testInput);
        boolean    value = evaluate(tokens);
        System.out.println(value);
    }
    // ...
}
// ...
~~~

Running this, we get:

~~~
true
~~~

It works! 🎉

What we did
---

After all that we finally have a working program that solves one small part of the problem! Our strategy was was to break down the problem into more manageable sub-problems which gave us a to-do list like this:

~~~
generate all truth tables given a formula
├── make sense of the input text
│   ├── tokenizing ✅
│   └── precedence rules
├── find every possible value for the variables 
├── evaluate all possible expressions from the formula
│   └── evaluate a single expression ✅
│       ├── "true", "false" equal themselves ✅
│       └── recursively traverse "and", "or", "not" ✅
└── print the results in the form a table
~~~

We looked through our to-do list of sub-problems until we got to something we could easily handle. Then we solved a single easy thing. Once we solved a couple of easy things, like tokenize and evaluate, we composed them together to solve a bigger part of the problem. Now we can work through the rest of the to-do list and hopefully apply the same strategy until we arrive at our goal.

Somehow that pattern seems very familiar...


End Remarks
---

Our program works well on its intending inputs: well-formed expressions. For example: `(and true false)` is a well-formed expression that we can evaluate to `false`. But what about "nonsense" expressions like `(true and or)` or `(and not)`?

With the way evaluate is currently defined, these return:

~~~
(true and or) == true 
// eval finds true, returns it, then stops consuming input

(and not) => Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException:
// eval keeps trying to consume input even after it runs out
~~~

This seems like the wrong thing to do. But that's okay (for now). As we move through the rest of our to-do list, we will keep these limitations in mind. At some point in solving the other sub-problems, we can fix this issue too.



If you read this far, then thanks for reading! See you next time when we extend this program further.

