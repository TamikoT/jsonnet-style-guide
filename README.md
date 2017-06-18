# Databricks Scala Guide

With over 1000 jsonnet files and templates, Databricks is to the best of our knowledge the largest user of Jsonnet. This guide draws from our experience coaching and working with engineers at Databricks.

Jsonnet is a language used most commonly to describe a finite number of complex, differentiated resources. For example, we may be describing services deployed within a Kubernetes cluster, differentiated by running in development versus production. As another example, we may be describing resources within a Cloud Provider, such as an Amazon RDS or Google Cloud SQL database, deployed across differnet regions.

Because we are most commonly describing a finite and somewhat fixed set of resources, it is useful to think of jsonnet as code which is executed at commit time (in the code repository sense) to materialize specific resources such as Kubernetes Deployment JSONs or AWS CloudFormation templates. In this way, the materialized templates are _production code_ and source code diffing tools are _unit tests_, which means that we can be more certain in the correctness of jsonnet templates without writing specialized tests.

Jsonnet is a relatively constrained language, but we have found that sometimes the most obvious way to build and extend jsonnet also leads to significant headache down the line for people to understand and extend code. We have found that the following guidelines work well for us on projects with high velocity -- depending on the needs of your team,  your mileage might vary.

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License</a>.


## <a name='TOC'>Table of Contents</a>

1. [Document History](#history)

2. [Syntactic Style](#syntactic)
    * [Naming Convention](#naming)
    * [Variable Naming Convention](#variable-naming)
    * [Line Length](#linelength)
    * [Rule of 30](#rule_of_30)
    * [Spacing and Indentation](#indent)
    * [Blank Lines (Vertical Whitespace)](#blanklines)
    * [Parentheses](#parentheses)
    * [Curly Braces](#curly)
    * [Long Literals](#long_literal)
    * [Documentation Style](#doc)
    * [Ordering within a Class](#ordering_class)
    * [Imports](#imports)
    * [Pattern Matching](#pattern-matching)
    * [Infix Methods](#infix)
    * [Anonymous Methods](#anonymous)

1. [Scala Language Features](#lang)
    * [Case Classes and Immutability](#case_class_immutability)
    * [apply Method](#apply_method)
    * [override Modifier](#override_modifier)
    * [Destructuring Binds](#destruct_bind)
    * [Call by Name](#call_by_name)
    * [Multiple Parameter Lists](#multi-param-list)
    * [Symbolic Methods (Operator Overloading)](#symbolic_methods)
    * [Type Inference](#type_inference)
    * [Return Statements](#return)
    * [Recursion and Tail Recursion](#recursion)
    * [Implicits](#implicits)
    * [Exception Handling (Try vs try)](#exception)
    * [Options](#option)
    * [Monadic Chaining](#chaining)

1. [Concurrency](#concurrency)
    * [Scala concurrent.Map](#concurrency-scala-collection)
    * [Explicit Synchronization vs Concurrent Collections](#concurrency-sync-vs-map)
    * [Explicit Synchronization vs Atomic Variables vs @volatile](#concurrency-sync-vs-atomic)
    * [Private Fields](#concurrency-private-this)
    * [Isolation](#concurrency-isolation)

1. [Performance](#perf)
    * [Microbenchmarks](#perf-microbenchmarks)
    * [Traversal and zipWithIndex](#perf-whileloops)
    * [Option and null](#perf-option)
    * [Scala Collection Library](#perf-collection)
    * [private[this]](#perf-private)

1. [Java Interoperability](#java)
    * [Java Features Missing from Scala](#java-missing-features)
    * [Traits and Abstract Classes](#java-traits)
    * [Type Aliases](#java-type-alias)
    * [Default Parameter Values](#java-default-param-values)
    * [Multiple Parameter Lists](#java-multi-param-list)
    * [Varargs](#java-varargs)
    * [Implicits](#java-implicits)
    * [Companion Objects, Static Methods and Fields](#java-companion-object)

1. [Testing](#testing)
    * [Intercepting Exceptions](#testing-intercepting)

1. [Miscellaneous](#misc)
    * [Prefer nanoTime over currentTimeMillis](#misc_currentTimeMillis_vs_nanoTime)
    * [Prefer URI over URL](#misc_uri_url)
    * [Prefer existing well-tested methods over reinventing the wheel](#misc_well_tested_method)



## <a name='history'>Document History</a>
- 2017-06-17: Initial version.

## <a name='syntactic'>Syntactic Style</a>

### <a name='lint'>Autoformatting</a>

Use `jsonnet fmt` to format files. This will fix basic style errors.


### <a name='variable-definition'>Variable Definition</a>

- Prefer `local` to `::` syntax for private/local variables. Unlike `::`, variables defined with `local` cannot be overridden by children, nor accessed by other files.
  ```
  {
    // CORRECT
    local myVariable = 3,
    result: myVariable + 1,

    // INCORRECT
    myVariable:: 3,
    result: $.myVariable + 1,
  }
  ```

### <a name='variable-naming'>Variable Naming Convention</a>

- Variables should be named in camelCase style, and should have self-evident names.
  ```
  local serverPort = 1000;
  local clientPort = 2000;
  ```

### <a name='linelength'>Line Length</a>

- Limit lines to 100 characters.
- The only exceptions are import statements and URLs (although even for those, try to keep them under 100 chars).

### <a name='linelength'>Indentation</a>
- Use 2-space indentation in general.
- Only method or class parameter declarations use 4-space indentation, to visually differentiate parameters from method body.
  ```
  local multiply(
      number1,
      number2) = {
    result: number1 * number 2
  }
  ```

### <a name='class-definitions'>ClassDefinitions</a>

- When defining a class, use the following syntax:
  ```
  local Animal(name, age) = {
    name: name,
    age: age,
  };
  {
    newAnimal:: Animal,
  }
  ```
- Above syntax is preferred as it is "type-safe" both in what parameters can be passed and what fields it exposes.
- Returning a dictionary with a "newXXX" method (rather than just returning the constructor directly) allows exposing constants, static methods, or related class constructors from the same file. In other words, it allows extending this class in the future without refactoring all downstream consumers.
- When defining a class with both required and optional parameters, put required parameters first. Optional parameters should have a default, or `null` if a sentinel value is needed.
  ```
  local Animal(name, age, isCat = true)
  ```
- Wrap parameter declarations by putting one per line with 2 extra spaces of indentation, to differentiate from the method body. Doing this is always acceptable, even if the definition would not wrap.
  ```
  local Animal(
      name,
      age,
      isCat = true) = { 
    name: name,
    ...
  }
  ```

### <a name='method-definitions'>Method Definitions</a>
- Method definitions follow the same syntax as class definitions.
- Methods defined within a class should always be defined with `::`, as they fail to render with `:`.
- Methods which return single values (rather than a dictionary) should use parentheses `()` to enclose their bodies if they are multi-line, identically to how braces would be used.
  ```
  {
    multiply:: function(number1, number2): (
      number1 * number 2
    ),
  }
  ```

### <a name='class-definitions'>Class Usage</a>
- Import all dependencies at the top of the file and given them names related to the imported file itself. This makes it easy to see what other files you depend on as the file grows.
  ```
  // CORRECT
  local Animal = import "animal.jsonnet.TEMPLATE";
  Animal.newAnimal("Finnegan", 3)

  // AVOID
  (import "animal.jsonnet.TEMPLATE").newAnimal("Finnegan, 3)
  ```
- Prefer using named parameters, one per line, when constructing classes or invoking methods, especially when they wrap beyond one line:
  ```
  // PREFERRED
  Animal.newAnimal(
    name = "Finnegan",
    age = 3,
  )

  // ACCEPTABLE, since it does not wrap
  Animal.newAnimal("Finnegan", 3)
  ```

### <a name='indent'>Spacing and Indentation</a>

- Put one space before and after operators
  ```
  local c = a + b;
  ```

- Put one space after commas.
  ```
  ["a", "b", "c"] // CORRECT

  ["a","b","c"] // INCORRECT
  ```

- Put one space after colons.
  ```
  {
    // CORRECT
    foo:: "bar",
    baz: "taz",
    { hello: "world" },

    // INCORRECT
    foo :: "bar",
    baz:"taz",
    { hello : "world" }, 
  ```

- Use 2-space indentation in general.

- Only method or class parameter declarations use 4-space indentation, to visually differentiate parameters from method body.
  ```
  // CORRECT
  local multiply(
      number1,
      number2) = {
    result: number1 * number 2
  }
  ```

- Do NOT use vertical alignment. They draw attention to the wrong parts of the code and make the aligned code harder to change in the future.
  ```
  // Don't align vertically
  local plus     = "+";
  local minus    = "-";
  local multiply = "*";

  // Do the following
  local plus = "+";
  local minus = "-";
  local multiply = "*";
  ```


### <a name='blanklines'>Blank Lines (Vertical Whitespace)</a>

- A single blank line appears:
  - Within method bodies, as needed to create logical groupings of statements.
  - Optionally before the first member or after the last member of a class or method.
- Use one or two blank line(s) to separate class definitions.
- Excessive number of blank lines is discouraged.


### <a name='doc'>Documentation Style</a>

- Use `//` for comments.
- Document parameters using a description, `@param`, and `@returns` similar to JavaDoc: 
  ```
  // Multicellular, eukaryotic organism of the kingdom Animalia 
  // @param name Name by which this animal may be called.
  // @param age Number of years (rounded to nearest int) animal has been alive.
  local Animal(name, age) = { ... } 
  ```
- Always put documentation at the top of each jsonnet file or template to indicate its purpose.

### <a name='file-structure'>File Structure</a>
- Jsonnet files which can be materialized with no further inputs should end with the ".jsonnet" suffix.
- Jsonnet files which requires parameters to be materialized or which are libraries should end with the ".jsonnet.TEMPLATE" suffix.
- Structuring libraries and imports is not a solved problem, so use your best judgement. In general if you have one common template and many individual instantiations, a workable pattern is:
  ```
  central-database/database.jsonnet.TEMPLATE <-- common template
  central-database/dev/database.jsonnet <-- dev instantiation which imports common template
  central-database/prod/database.jsonnet <-- prod instantiation which also imports common template
  ```

## <a name='lang'>Best Practices</a>

### <a name='golden_pattern'>The Golden Pattern</a>

In most cases, you can define a single class which outputs the entire template you're looking for, and have a single concrete jsonnet file per actual resource to create. For example, you might define a common database template and a single dev jsonnet and a single prod jsonnet which imports from the common template.

In this case, a pattern like the following is most preferred:

```
// Common database template for an Amazon RDS database to store users (users-database.jsonnet.TEMPLATE)
local UsersDatabase(instanceSize, highAvailability) = {
  ... CloudFormation stack describing an AWS RDS ...
  rdsInstanceSize: instanceSize,
  multiAvailabilityZone = highAvailability,
  ... etc ...
};

{
  newUsersDatabase:: UsersDatabase
}


// Dev users database (in dev/users-database.jsonnet)
local UsersDatabase = import "../users-database.jsonnet.TEMPLATE";
UsersDatabase.newUsersDatabase(
  instanceSize = "db.m4.medium",
  highAvailability = false,
)


// Prod users database (in prod/users-database.jsonnet)
local UsersDatabase = import "../users-database.jsonnet.TEMPLATE";
UsersDatabase.newUsersDatabase(
  instanceSize = "db.m4.large",
  highAvailability = true,
)
```

Use this pattern as far as it will get you. Avoid implementing further abstractions and avoid default parameters for as long as possible. Keeping the number of abstractions low usually makes templates easier to understand. Avoiding default parameters means you have to explicitly choose a value in every situation (reducing corretness bugs).

On the flip side, new additions may require significant mechanial work due to repition. Templates __do not have to be DRY__ (don't repeat yourself) because they are fully materialized at commit time, so correctness issues of repetitiveness are reduced and readability is more important. Use your best judgement when deciding when to build out a new abstraction to avoid repetition.

### <a name='parameter_objects'>Paramter objects</a>

In situations where sets of parameters are shared between multiple templates or objects, define parameter objects which extract out the common set.

```
local AwsVpcParams(region, accountId, virtualNetworkId, encryptionKeyId) = {
  ...
};


local UsersDatabase(awsVpcParams, instanceSize, highAvailability) = {
  ...
}

local FrontendVirtualMachine(awsVpcParams, instanceName) = {
  ...
} 
```

### <a name='overriding_fields'>Overriding and inserting fields</a>

Jsonnet has excellent support for supplementing existing JSON structures with new fields, for example:

```
local protoCat = Animal.newAnimal(name = "Finnegan", age = 3);
// ... many lines later ...
local myCat = protoCat + {
  catYears: 28,
}
```

This pattern is very convenient for implementing arbitrary transformers on data strutcures, but it should be used with caution because it makes it hard to reason about how code within the parent class is executing and what fields are provided on "myCat".

This becomes especially confusing if you have multiple layers of such additions, or helper functions with transform properties within the input. In the rare cases that parameters are determined at various points within the code, prefer to make helper functions with construct and mutate parameter lists, which are ultimately passed to an Animal object.

For example:

```
// Better, but still to be avoided
local protoCat = Animal.newAnimalParams(name = "Finnegan", age = 3);
// ... many lines later ...
local myCatParams = protoCat + {
  catYears: 28,
};
local myCat = Animal.newAnimal(myCatParams)
``` 

This pattern ensures that inputs and outputs are fully determined by the code within Animal.

