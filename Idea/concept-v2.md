# Overview
-----------
My project for GSOC 2019 is `Handle Nested Programming Language`. 

## Why is this necessary
--------------------------
A source code generally consists of a single programming languages. But there are also many cases when we encouter a file which has more than one programming langauge embedded into it.For eg: Python in Jinja2, HTML in Jinja2, reStructured Text ( can have different code blocks), different other web templating languages.

As of now coala can only provide static analysis tool to a file which has only one programming language. This project would enable coala to deal with files that have nested programming language and provide with proper static analysis for such file. That means, by the completion of this project coala would be able to recognize appropriate language of various parts of the file and provide meaningful analysis to it. 

The aim of this project is to provide an interface for coala, so that it would be able to lint a file with nested languages. This also means providing interface for bear writers to write the analysis tools as they generally do for a particular language.

# Approach
-----------

There are two approaches that I have encountered to solve the problem.

## Approach 1
-------------
This approach deal with seperating out a nested programming language into seperate language file. The steps for this include.

1. The user is asked for the combination of languages that is being used.

2. The file is then converted into a valid format so that we get so lint errors. By valid format I mean, when two languages are nested upon each other there is a high chance that the variable declared in one language is depends on another.
For eg:
```
{% for var in var_list %}
from {{var}} import x
{% endfor %}

```
As you see here the value of `{{var}}` ,depends on `var_list` provided from Jinja2. If we seperate the file as it is then the line present in python would turn out to be a error. So we convert the file to the valid format.

We can do that by removing `{{ var}}` from the line and replacing it by `j_var` so that it looks like a valid line.

3. The valid-format file is then seperated into corresponding language and then linted seperately.
4. The linted files are then merged back together.


But on [futher research](https://docs.google.com/document/d/1vTb3xZQ7_-290o6wVjFa0EKvC2Ppbr9UkXLNeYiMtDw/edit?usp=sharing) and working, I found that this would not be a proper approach to the problems.

The Reasons for this are as stated below:
-----------------------------------------

1. The above approach requires us to produce extra files and then lint them sperately. This would only lead in taking up more time as the linting needs to be done one after another.

2. Seperating the files would mean that we would have to maintain the position of the changes that occur in each of the file.

3. Once the changes are made it also becomes hard to combine the file back to the exact location.

4. The above approach also provided a straight out way to lint files. It provides very less abstraction layer in order to accomodate other languages.

Hence because of the above reason I decided that it is futile to sperate the files into different files and lint them seperately. Thus I tried developing another approach which is explained below.

## Approach 2
--------------

This approach would be similiar to how coala generally works with single language file. Here we would be following a concept called as `Bears loading Bears`. This concept is new to coala and hence have to be designed in this project for this to work.

The following image shows the first abstraction layer of the process:

IMAGE 1 - ABSTRACTION LAYER

I would be explainig the entire approach as step by step as we go ahead.

For the sake of simplicity, let's consider we have a Jinja2 file with python Embedded in it, which is generally the case of the templates coala designs in `coala-moban`. Let the name of the file be `test.py.jj2`

### Step 1
----------
>> The user provides coala the file along with an argument `--nested-language`. This argument would then launch coala in Nested langauge mode.

The following steps occur here:

1. `coala -files test.py.jj2 --nested-language`

2. coala then asks the user to choose the kind of file it is. ( As we know there are very few languages which have nested languages so the list would be short)

```
     test.py.jj2
**** NESTED LANGUAGE MODE ****
!    ! Please Select the type of Language
[    ]  1. Jinja2 and python
[    ]  2. Jinja2 and HTML
[    ]  3. Restructured Text
[    ]  4. PHP and HTML
[    ] Enter number (Ctrl-D to exit): 1
```

3. coala would then ask for bears that the user wishes to run on each language.

```
     test.py.jj2
**** NESTED LANGUAGE MODE ****
!    ! Please Select the Bears you wish to run on Jinja2.
!    ! Enter the Bears with comma(,) to seperate them

[    ] Enter Bears (Ctrl-D to exit): Jinja2Bear
```
```
     test.py.jj2
**** NESTED LANGUAGE MODE ****
!    ! Please Select the Bears you wish to run on Python.
!    ! Enter the Bears with comma(,) to seperate them

[    ] Enter Bears (Ctrl-D to exit): PEP8Bear,PyImportSortBear,PyLintBear
```

### Step 2
-----------

This is where we talk about `Mechanism 1`.

Now that the file is loaded and the Bears to be executed are taken into account. The `Mechanism 1` kicks in.

IMAGE OF NECHANISM 1

The Role of `Mechanism 1` is as follows:

1. It loads up all the bears mentioned by the user. It is very important to note that we maintain only one Instance of each bear. The reason for this would be explained in a little time.

2. Analyze the language.

3. Pass the line along with language to the `ListOfBear` Class.

This would also act as an interface for new Bear writers. They could use all the classes and the functions created in the process of this step and write bears for Nested Language.

Loading up of bears 
-----------------------

It is very important to understand that we maintain only one instance of each bear that is accessible by `ListOfBear` class. The reason we say we need only one Instance of bear is because, for a meaningful analysis of a code the bear should have the knowledge of what's in the entire code file,rather than only on the particular line.

For example: In `Jinja2Bear` for the Bear, maintains a stack of all the control labels so that it would know if there are any missing control labels. The bear also has an identation feature(recent PR made my me, gives this feature to the bear) also maintains a stack for proper indentation. This can only happen if we have one single Instance of bear because other wise a new stack would be getting created everytime the bear is run.

Analysing the Langauge
------------------------

This is the next step. This mechanism takes in line by line. And analyses the language.

```
from coalib.language_parsers import parsers

# Get the appropriate language paraser
parser = parser.get_parser(user.selected_language)
analyze_language = parser.analyze_langauge()

# Analyze the language
for line in lines:
	orig_line = line
	language,valid_line = analyze_language(line)
	pass_to_bears(orig_line, valid_line, language)

```

It is important to note here.

That we would be `creating specialized AST parsers for each combination of language` and store them in `language_parsers`

Each specilized AST parser would have a function named `analyze_language()` which would return the language after parsing through them. This function also converts a particular line to a valid format.

For eg:
The line `from {{var}} import x` when passed to `analyze_language` returns `from j_var import x`

The way in which the above step works is:

1. We use the value of selected langauge to choose the appropriate parser.
2. We then use the `analyze_language` specific to each parser to identify the language.
3. The line is analyzed and then converted to a valid format and then returned. 
4. The `orig_line`,`valid_line` and `language` is passed on to the `ListOfBear` class. The reason we have the orig_line is because the line being displayed in the diff should be the original line instead of the valid_line. It is important to note that the linting is still done on `valid_line` only but once the linting is done, the line get's converted back to `orig_line` ( WORK NEEDS TO BE DONE - How can the valid_file and orig_file be combined once the linting is done. This means there has to be change before the diff will be applied. Have to intervene in the applying diffs of Bears. Is IT possible?)

### Step 3
-----------

`ListOfBear` class is the class which would pass on the lines to the actual Bears.

This is where the actual linting going to occur. This is where we would be actually using the concept of `Bears Loading Bears`. 

The Role of this class would be:

1. Use the details provided by the `Mechanism 1 ` to pass the line to respective bears intialized in the `Mechanism 1`.

2. Get the diff from the bears and then convert the `valid_line` to `orig_line` form.

A  sample mockup of this bear would be:

```
lang_bear = []

# Collect the bears associated with each language
for lang in get_user_selected_language_list:
	bears =  get_bears_for(lang)
	lang_bear.append({lang : bears})

# Lint the line by passing it to associated bears
# Gather the diff and change it accordingly before we merge.
for langs in lang_bear:
	for lang,bears in lang.items():
		for bear in bears:
			generated_diff = lint_line(orig_line, valid_line, bear)
			handle_diff(generated_diff, orig_line, valid_line)

```

The main challenge in this step for now is to divert the diff from bears to here and then change the diff accordingly and then apply the diff.

And this is the approach that I think would work.

To re-iterate these steps we have:
------------------------------------
1. Take in all the information from the user.
2. Analyze the lines of the langauges -  Will be done using parsers - Different parser for different combination
3. Convert the line to valid format.
4. Pass the line to respective bears of each languages.
5. Change the diff to match the real line.

Things I would have to work on:
--------------------------------
1. Creating parsers for differnt languages.
2. Creating an Interface on top of `Mechanism 1` so that bear writers use it to write normal analysis.
3. Find a way to load an Instance of bear from another bear/Class such that only one Instance in maitained overall
4. Find a way to divert the diff so that you can change the line back to original format.
5. Display the diff from elsewhere.