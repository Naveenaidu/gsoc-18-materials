My target for the next GSOC 2019 is `Handling Nested Languages`. Though I do not yet have a solution to actually solve the project. TBH I don't
know, How am i going to solve it. 

My plan as of now, is to work on a subproblem - `Linitng Python in Jinja2`. I could then build upon it to support other languages. 

How am I proceeding with the subproblem:

The above mentioned problem can be solved in 3 steps:-
1. Parse through the Jinja2 file and then emit `.py` files.
2. Use this generated file to lint.
3. Lint the file and show the results.

The Problems:
1. How do i parse the files? 
2. How should the output actually look like
3. How could i then add the corrected file/linted file back into jinja2 orignal file
4. How do i integrate `coala` with all this.

#h1 The Idea:-

#h2 A. Parsing the file
=======================

During my reasearch, I came across a blog which explains how `coverage.py` works. And it is from there I developed few ideas:-

1. There are two ways in which parsing can be done. `Interpretation model` and `Compilation Model`.

```
Interpretation model:
~~~~~~~~~~~~~~~~~~~~

This model parses through the entire file and produces a data structure which represents the template.This data structure is then rendered by assembling the values to the data structure.

Django templates,uses this model.
```

```
Compilation Model:
~~~~~~~~~~~~~~~~~~

This model parses through the template and generates some form of directly executing code. The rendering phase then executes this code, but supplying the input to the compiled code.

Jinja2 uses this model.
```

IMHO, the second model is faster and better suited when we deal with the cases where a particular template has to be used many times. The reason being, that it's faster to use a already compiled template and supply values to it rather than generating a data structure and then rendering it. The former saves a lot of time. And the `coverage.py` also uses the former method to produce html templates for it's coverage documentation.

Now let's get along with the actual parsing part.

Before we go on we need to have an idea about how the ouput would look like.

Let's assume our Jinja2 template looks something like:

```
import sys

{% for var in var_list %}
from {var} import x
{% endfor %}

def function:
{% if x in number %}
	x = 'a'
{% endif %}
```

The generated `output.py` file after we run parser should look something like:

```
import sys

from j_var import x

def function:
	x = 'a'
```

There are few points here that needs to be kept in mind before we start parsing the file:

	1. The indendation is very important. The indentation should be preserved as it is in the original file
	2. The local variables and the global variables will be prefixed by `j_` so that they don't collude with the variables we use.
	3. Exclude the lines that begins with `{` and ends with `}`

#h4 An approach to solve the issue:
===================================

We could have a `CodeGenerator` class, whose main task is will be to:
	1. Parse the file
	2. Store all the local and global variables
	3. Add all the python lines to a list and also store the information needed to merge the code back to jinja file.
	4. Do not print any jinja2 statements.
	5. Print the python code from the list at the appropriate line number

`CodeGenerator` Mockup:
=======================

```
def __init__(self, indent=0):
        self.code = []
        self.indent_level = indent
        self.loops_variable = []
```
```
# Add a line of source to the code.

def add_line(self, line):
        
    self.code.append([line, "\n"])
```

```
# Get all the global variables from the template.

def get_global_variables(self):
		# Get the Python source as a single string.
        python_source = str(self)

        # Execute the source, defining globals, and return them.
        global_namespace = {}
        get_global_variable(python_source,global_namespace)

        return global_namespace
	
```

Now let's add the functions that parses through the code, to store the local variables and also removes the blocks related to Jinja2.

For this we first need to get the `tokens` of the jinja2 language, and then use these token to seperate the python code from jinja2. We can use `regex` to find the tokens.

Once the text is split into tokens like this, we can loop over the tokens, and deal with each in turn. By splitting them according to their type, we can handle each type separately.

eg:- The `if` tag should have a single expression, so the words list should have only two elements in it.We push `if` onto `a stack` so that we can check the `endif` tag. The expression part of the if tag is compiled to a Python expression with _expr_code, and is used as the conditional expression in a Python if statement._


```
def get_local_variables(self,source_code):
	tokens = re.split(r"(?s){%.*?%}|{#.*?#})", source_code)

	for token in tokens:
		if token.start_with('#'):
			continue
		
		elif token.start_with('%'):
			words = token.split(' ')

			if words[0] == 'if':

				if len(words)!= 2:
					print("ERROR: Invalid For loop")
				stack.append('if')

			elif words[o] == 'for':
				if len(words) != 4 or words[2] != 'in':
                        print("Don't understand for", token)

                stack.append('for')

                self._variable(words[1],self.loop_vars)

            # check for proper closing tags    
            elif words[0].starts_with("end"):
            	end_what = words[0][3:]
            	start_what = stack.pop()   

            	if start_what != end_what:
            		print("ERROR: Mismatched Tags")     

```



Here `_variable` is a method which get's the local variable from the loop and adds it to the data type we provide.
PROBLEM SPOTTED:

Now let's create a function that will generate the actual `.py` files
```
def generate_output_python_file(self,source_code):
	
	template_content = source_code

	global_variables = get_global_variables(self)

	for line in template_content:
		if line.start_with('from'|'import'):
			add_line(line)

		elif line.start_wihr('{'):
			continue

		elif:

			if line.contains('{{'):

				templete_local_variable = line.extract_variable()
				python_variable = add_prefix(template_local_variable,'j_')
				line = replace_variable(line,template_local_variable,python_variable)

				add_line(line)


```

The above function does the following things:
1. Get's the template file
2. Get's all the global variables
3. Check if a line starts with only plain words( it's means that it's a python code) then add as it is
4. If a line starts with `{` ( which means it's a template file related code) - ignore it
5. If a line contains `{{ var }}`( which is used to get he value) .. replace the `var` by `j_var`


This is how, I have decided how to parse the Jinja2 file to convert it to a python file. 
---------------------------------------------

INCOMPLETE:

How will you know that a varibale from a `for` loop belongs to a particular loop. But why do we need to think about it. We just need to store the `local_variable` so that we can append a prefix `_j` to it.

There is a problem here. The output file we get and the `generated file` from the `compilation model` is different. In compilation model the lines encoded in ` {}` are stripped of those and added into normal file. But we do not want that.( Refering from the 500 lines to generate Template Engine). That was done because they wanted to generate a template. And hence they required it to be a compilable code. We do not need that, we can just ignore the lines and go ahead.

All we would need is to store the local variable and ignore the other lines.

I mean like go into a particular for loop and check if the local variable of the for loop is being used somewhere and then  replace that local variable by prefixing `j_`, so that linter would not have a problem reading that.