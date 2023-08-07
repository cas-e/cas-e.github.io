Generating Truth Tables in Java : Part 3 : Tables
===

This the final part in a three-part series, the start of which is [here](java-bool-expr.html).

The full code for this blog post can be found [here on github](https://github.com/cas-e/blog-code/tree/main/JavaBoolTruthTable).

So Far...
---

In the first post we figured out how to evaluate simple boolean expressions. Then, in the next post we finished up dealing with our input text. Our to-do list now looks like this:

~~~
generate a truth table when given a formula
├── make sense of the input text ✅
├── find every possible value for the variables 
├── evaluate all possible expressions from the formula
│   └── evaluate a single expression ✅
└── print the results in the form a table
~~~

Let's get started on finding every possible value for the variables.


Finding All the Values
---

Here's a demo class of how that will look under a file called Demo.java:

~~~
public class Demo {
	public static void main(String args[]) {
		generate(1);
	}
	public static void generate(int n) {
		genVals(new boolean[n], 0);
	}
	public static void genVals(boolean[] bs, int n) {
		if (n == bs.length) {
			for (boolean b : bs) { System.out.print(b + " ");} // (1)
			System.out.println();
			return;
		}
		bs[n] = false;
		genVals(bs, n+1);
		bs[n] = true;
		genVals(bs, n+1);
	}
}
~~~

Running this demo as it is, with a call to `generate(1)`, should produce every possible value for the boolean expression of just one variable `x`. And it does: 

~~~
false 
true
~~~

And if we had an expression with two variables like `x && y`, we would need to consider all the inputs from `generate(2)`:

~~~
false false 
false true 
true false 
true true 
~~~

It's a good time to note, if it's not already obvious, that these are appearing in the order in which we count binary numbers `00`, `01`, `10`, `11`. This isn't the only order we could traverse the space of possibilities (we could, for instance, use the [Gray code order](https://en.wikipedia.org/wiki/Gray_code)). But "counting order" is the one commonly used for tables, and for, uh, counting. So it's the one we use. 

By the way, when I was researching for this post, the above algorithm came up often under articles with titles like, "How to GIT GUD and CRUSH your FAANG interview with BINARY PERMUATIONS". So, if the appeal of printing tables isn't intrinsically satisfying for it's own sake, maybe study that algorithm anyways for job interview reasons. (Just kidding, **of course** printing tables is intrinsically satisfying!)

Anyways, let's look at `(1)` in that code one more time. Right where we print out one line of boolean values from that array, is exactly the time we have all the information needed to make one call to our `evaluate` method and get the answers. For instance with `x && y`, where we need `generate(2)`:

~~~
false false // evals to -> false
false true  // evals to -> false
true false  // evals to -> false
true true   // evals to -> true
~~~

So we can use this algorithm to both: a) find the variable values, and b) invoke `eval` to find all the answers. We can then just pass all of that info along to some printing routine.

But as of right now, our evaluator can't handle any variables, so we will make some modifications there.

Extracting Variables from Expressions
---

First, we need to determine how many variables we have in an expression and what their names are. An expression like `x && x || x` may be a little long but it only has one variable: `x`, and will have a correspondingly small table: either `x` is true or `x` is false, so two options with two evaluations. 

So we need to extract the variables into a collection, and we also need to remove any duplicate variable names. Extracting variables will be a good job for the Tree class from last time, since determining variables is a syntax kind of thing.

We will allow any name to be a variable as long as it is not one of our keywords like `&&` or `true`. (This has the terrible consequence that the tab character is a valid variable name-- if the number of users of this program were to grow past 1, we would need to update  `tokenize` to ensure this wasn't possible. For now, we allow it.)

So in Tree.java we define our keywords:

~~~
private static String[] keywords  = {"true", "false", "&&", "||", "!", "(", ")"};
~~~

And we  define a method to check if something is a member of our set of keywords:

~~~
private static boolean member(String s, String[] els) {
	for (String e : els) { if (s.equals(e)) return true; }
	return false;
}
~~~

Our task now is to look at an array of our tokens, filtering out the tokens that are keywords and removing any duplicate variables. And it just so happens that Java's streams provide exactly these methods as `filter()` and `distinct()`:

~~~
private static String[] getVars(String[] tokens, String[] keywords) {
	return Arrays.stream(tokens).distinct()
		.filter(x -> !member(x, keywords)).toArray(String[]::new);
}
~~~

Finally we wrap this `getVars` method up in another public method for use by our main TruthTable class:

~~~
public static String[] variables(String input) {
	return getVars(tokenize(input), keywords);
}
~~~

Which is called like:

~~~
String[] vars = Tree.variables("x && y || x") 
// vars is ["x", "y"]
~~~

We now have one method for booleans all, one method for vars to find them. We will write one method to scan them all, and in a HashMap bind them. 


Binding Boolean Values to Variables
---

Our evaluator for boolean expressions currently accepts a `Node` tree representing an expression. It now needs to additionally accept a mapping from variable names to boolean values. This kind of mapping is often referred to as an "environment". We need it to have one method: `get(String s)` that returns the corresponding boolean. And a `java.util.HashMap` has this built in for us.

First, working in our main TruthTable class, lets define a method to construct new environments...

~~~
public static HashMap<String, Boolean> makeEnv(String[] keys, boolean[] vals) {
	HashMap<String, Boolean> env = new HashMap<>();
	for (int i = 0; i < keys.length; i++) {
		env.put(keys[i], vals[i]);
	}
	return env;
}
~~~

The evaluator has one extra line now, and one extra parameter, passed to all its recursive calls:

~~~
public static boolean eval(Tree.Node n, HashMap<String, Boolean> env) {
	if (n.isVar)               return env.get(n.val);
	if (n.val.equals("true"))  return true;
	if (n.val.equals("false")) return false;
	if (n.val.equals("!"))     return !eval(n.left, env);
	if (n.val.equals("||"))    return eval(n.left, env) || eval(n.right, env);
	if (n.val.equals("&&"))    return eval(n.left, env) && eval(n.right, env);
	System.out.println("unrecognized token in evaluator");
	System.exit(1);
	return false;
}
~~~

That gives us everything we need to evaluate the expressions from the `generate` method. We get all the variable possibilities and all the evaluations from `generate`, and we just need to pick a data structure to store them so we can print it as a grid.

Data Return Type for the Generate Method
---

The first choice that came to my mind was a nested array, `String[][]`. However, that ended up complicating the printing routine more than necessary. In my opinion, it turns out simpler for printing if we just keep all the data flat as a single array `String[]`. Of course, then we lose information about the grid structure. But, as long as we pass along a single `int` representing the row length (or equivalently, the number of columns), then we can recover the grid shape. 

We make another wrapper around a basic array with some added info and some helpers. Let's call it `Seq` for sequence, much like in the other posts. 

~~~
static class Seq {
	int index = 0;
	int rowLength;
	String[] array;
	public Seq(String[] ts)     { array = ts; }
	public String next()        { return array[index++]; }
	public void puts(String s)  { array[index++] = s; } // (1)
	public boolean more()       { return (array.length - index) > 0; }
	public void resetIndex()    { index = 0;} // (2)
	public boolean endOfVars()  { return (rowLength-1) == index%rowLength; } // (3)
}
~~~

In `(1)`, we have a method `puts` that we can think of as "put string", that just writes into the array. Once we `puts` all our strings representing booleans into the array, `resetIndex()` at `(2)` sets the index counter to it's original state so that the printing routing gets a fresh `Seq` to iterate through. As for `(3)`, that is how we recover the grid structure through the saved `rowLength`. Note though, in the future printing routine, it's more useful to know the `endOfVars()` which is one index before the actual end of the rows. To see why, let's remember our table end goal:

~~~
x     y     | formula
---------------------
false false | false
false true  | false
true  false | false
true  true  | true
---------------------
~~~

We will use the `endOfVars()` method to determine where to put the vertical bars between the last variable value on a row and the formula value.

And I do apologize for referring to future code we haven't written yet, but I do hope everything becomes clear by the end of the post.

On that note I think we are ready to `generate` all our values

Generating Values
---

The `generate` method here is similar the one we defined in the beginning of this post, but now it stores its data instead of printing it, and additionally invokes the `eval` method on the data, passing along the results of that as well.

First we define a wrapper method around `generate` that will set up the `Seq` to store the results.

~~~
public static int twoToThe(int x) { return 1 << x ;}
public static Seq evaluations(Tree.Node expr, String[] vars) {
	int outputSize = (vars.length+1) * twoToThe(vars.length);
	Seq output = new Seq(new String[outputSize]);
	output.rowLength = vars.length + 1;
	generate(expr, vars, new boolean[vars.length], output, 0);  // (1)
	output.resetIndex();                                        // (2)
	return output;
}
~~~

Notice that since the default Java arrays are not resizable, we do some calculations to obtain the right size array before sending the whole Seq into `generate`. At (1) we also call reset so that the next method in the pipeline can iterate over the results. And we can see already from the call to `generate` at `(1)` that five parameters! That is towards the upper limit of how many params I would like to put in a method, but it is useful since `generate` is really driving the whole process:

~~~
public static void generate(Tree.Node expr, String[] vars, boolean[] bools, Seq s, int n) {
	if (n == vars.length) {
		for (boolean b : bools) { s.puts("" + b); }
		boolean evaluated = eval(expr, makeEnv(vars, bools));
		s.puts("" + evaluated);
		return;
	}
	bools[n] = false;
	generate(expr, vars, bools, s, n+1);
	bools[n] = true;
	generate(expr, vars, bools, s, n+1);
}
~~~

This is just like the demo from before, but we `puts` all our booleans into the output `Seq` instead of printing them. At which point, we zip them up with the variables into a HashMap, and send that and the expression tree along to the evaluator. Then we can add the resulting boolean to the output sequence as well. 

And as it stands right now, we could run this...

~~~
public static void main(String[] args) {
	String   input = "x && y";
	Tree.Node expr = Tree.parse(input);
	String[]  vars = Tree.variables(input);
	Seq     values = evaluations(expr, vars);
	for (String t : table.array) { System.out.print(t + " ");}
}
~~~

Which results in:

~~~
false false false false true false true false false true true true
~~~

Which is correct! But hard to read. That's a formatting issue for the next step in the project. For now we cross two more items off our list. 

~~~
generate a truth table when given a formula
├── make sense of the input text ✅
├── find every possible value for the variables ✅
├── evaluate all possible expressions from the formula ✅
└── print the results in the form a table
~~~

Formatting 
---


Before we start writing the code, let's take another look at the kind of formatting we are aiming for. For `x && y` we want:

~~~
x     y     | formula
---------------------
false false | false
false true  | false
true  false | false
true  true  | true
---------------------
~~~

In the above, we add a padding space after `"true"` to make it line up nicely with `"false"` values. It all works nicely here with that small modification, but what if the variable names are much longer? For an expression like `myBoolean1 && myBoolean2`:

~~~
myBoolean1 myBoolean2 | formula
-------------------------------
false      false      | false
false      true       | false
true       false      | false
true       true       | true
~~~

Here, we have to add many padding spaces to both `"false"` and `"true"` in order to get things to line up nicely. The amount of padding depends on the length of the longest variable name, or `"false"`, whichever is greater. So we need some methods to determine what the longest name is, and some method that can add a variable amount of spaces: 

~~~
public static int lengthiest(String[] vars) {
	return java.util.Arrays.stream(vars).map(x -> x.length())
	             .reduce("false".length(), (x, y) -> (x > y) ? x : y);
}
public static StringBuilder pad(StringBuilder sb, String a, int l) {
	return repeat(sb, " ", (l - a.length()) + 1);
}
public static StringBuilder repeat(StringBuilder sb, String s, int n) {
	for (int i = 0; i < n; i++) { sb.append(s); }
	return sb;
}
~~~

Okay, now we can get started on the routine that does the rest. The first thing we will do is cover the special case where someone just enters a boolean expression without any variables at all. All that can be done then is to evaluate the one expression and print either `"true"` or `"false"`. We also pass the evaluator an empty HashMap, since it needs all of it's arguments. 

~~~
public static String tabulate(Tree.Node expr, String[] vars, Seq grid) {
	StringBuilder sb = new StringBuilder();
	if (vars.length == 0) {
		boolean single = eval(expr, new HashMap<String, Boolean>());
		return sb.append(single).append("\n").toString();
	}
	// ...
~~~

Now the rest of the code beyond that point is dealing strictly with printing the tables. Here's the full method:

~~~
public static String tabulate(Tree.Node expr, String[] vars, Seq grid) {
	StringBuilder sb = new StringBuilder();
	if (vars.length == 0) {
		boolean single = eval(expr, new HashMap<String, Boolean>());
		return sb.append(single).append("\n").toString();
	}
	String form = "| formula";
	int longest = lengthiest(vars);
	int width = vars.length * (longest + 1) + form.length();
	sb.append("\n");
	for (String a : vars) { pad(sb.append(a), a, longest); }
	sb.append(form).append("\n");
	repeat(sb, "-", width).append("\n");
	while (grid.more()) {
		String b = grid.next();
		pad(sb.append(b), b, longest);
		if (grid.endOfVars()) sb.append("| " + grid.next() + "\n");
		
	}
	repeat(sb, "-", width).append("\n");
	return sb.toString(); 
}
~~~

And there it is! In the first post in this series, we said the program should be able to handle longer cases like `p || q && !r && q`. And if we run the following:

~~~
public static void main(String[] args) {
	String   input = "p || q && !r && q";
	Tree.Node expr = Tree.parse(input);
	String[]  vars = Tree.variables(input);
	Seq     values = evaluations(expr, vars); 
	String   table = tabulate(expr, vars, values);

	System.out.print(table);
}
~~~

The output is:

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

It's working! Well, it certainly seems to. Testing will have to be left as an exercise for the reader. (I have only tested many cases on my end, nothing exhaustive.) For now, we can update our list:

~~~
generate a truth table when given a formula
├── make sense of the input text ✅
├── find every possible value for the variables ✅
├── evaluate all possible expressions from the formula ✅
└── print the results in the form a table ✅
~~~

In fact, we can triumphantly collapse this whole list and consider this project finished. All that's left is one last satisfying checkmark:

~~~
generate a truth table when given a formula ✅
~~~


