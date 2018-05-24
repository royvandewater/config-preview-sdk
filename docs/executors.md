# Authoring and using Executors 
- [What _is_ an executor?](#what-is-an-executor)
- [Common uses of executors](#common-uses-of-executors)
- [Declaring and using `executors` in an orbs.](#declaring-and-using-executors-in-an-orbs)
- [Overriding keys when invoking an executor](#overriding-keys-when-invoking-an-executor)
- [Using parameters in executors](#using-parameters-in-executors)

## What _is_ an executor?
Executors define the environment in which the steps of a job will be run. When declaring a `job` in CirlceCI configuration you tell it what type of execution environment to run in (`docker`, `machine`, `macos`. etc.) and other parameters of that environment such as environment variables to populate, which shell to use, what size `resource_class` to use, and so on. Executor declarations in config outside of jobs can be used by all jobs in the scope of that declaration, allowing you to reuse a single executor definition across multiple jobs.

An executor definition includes the subset of the children keys of a `job` declaration related to setting the environment for a job to execute. That subset is one or more of the following keys:

- `docker` or `machine` or `macos` 
- `environment`
- `working_directory`
- `shell`
- `resource_class`

A simple example of using an executor:

```
version: 2
executors:
  my-executor:
    docker:
      - image: circleci/ruby:latest

jobs:
  my-job:
    executor: my-executor
    steps:
      ...
```

In the above example the executor `my-executor` is passed as the single value of the key `executor`. Alternatively, you can pass `my-executor` as the value of a `name` key under `executor` -- this method is primarily employed when passing parameters to executor invocations (see below):

```
jobs:
  my-job:
    executor:
      name: my-executor
    steps:
      ...
```

## Common uses of executors
Executors in configuration were designed to enable:
1. Reusing a defined execution environment in multiple jobs in _config.yml_.
2. Allowing an orb to define the executor used but all of its commands, allowing you to use the commands of that orb in the execution environment defined by the orb's author.

### Example - using an executor declared in config.yml in many jobs
Imagine you have several jobs that you need to run in the same Docker image and working directory with a common set of environment variables. Each job has distinct steps, but should run in the same environment. You have three options to accomplish this:

1. Declare each job with repetitive `docker` and `working_directory` keys.
2. Use YAML anchors to achieve some reuse.
3. Declare an executor with the values you want, and invoke it in your jobs.

With an executor declaration your configuration might look something like: 

**Without executors**
```
jobs:
  build:
    docker:
      - image: clojure:lein-2.8.1
    working_directory: ~/project
    environment:
      MYSPECIALVAR: "my-special-value"
      MYOTHERVAR: "my-other-value"
    steps:
      - checkout
      - run: echo "your build script would go here"
      
  test:
    docker:
      - image: clojure:lein-2.8.1
    working_directory: ~/project
    environment:
      MYSPECIALVAR: "my-special-value"
      MYOTHERVAR: "my-other-value"
    steps:
      - checkout
      - run: echo "your test commands would run here"
```
    
**Same Code, With executors**
```
executors:
  lein_exec:
    docker:
      - image: clojure:lein-2.8.1
    working_directory: ~/project
    environment:
      MYSPECIALVAR: "my-special-value"
      MYOTHERVAR: "my-other-value"
    
jobs:
  build:
    executor: lein_exec
    steps:
      - checkout
      - run: echo "hello world"
      
  test:
    executor: lein_exec
    environment:
      TESTS: unit
    steps:
      - checkout
      - run: echo "how are ya?"
```

## Declaring and using executors in an orbs.
In orbs you can declare individual executors as files in an `executors` folder of the orb. The file name of your executor is the executor name.

**Directory structure**
```
- orbs
  - foo-orb
    - executors
      - bar.yml
  - baz-orb
    - executors
      - bar.yml
```

**Calling the above in your build config**
```
jobs:
  some_job:
    executor: foo-orb/bar  # prefixed executor
  some_other_job
    executor:
      name: baz-orb/bar    # prefixed executor
```
Note that `foo-orb/bar` and `baz-orb/bar` are different executors. They both have the local name `bar`, but are independent executors living in different orbs.

## Overriding keys when invoking an executor
When invoking an executor in a `job` any keys in the job itself will override those of the executor invoked. For instance, if your job declares a `docker` stanza, it will be used, in its entirety, instead of the one in your executor.

There is **one exception** to this rule: `environment` variable maps are additive. If an `executor` has one of the same `environment` variables as the `job`, the `job`'s value will win. For example, if you had the following build configuration:

```
executors:
  python:
    docker:
      - image: python:3.7.0
      - image: rabbitmq:3.6-management-alpine
    environment:
      ENV: ci
      TESTS: all
    shell: /bin/bash    
    working_directory: ~/project

jobs:
  build:
    docker:
      - image: python:2.7.15
      - image: postgres:9.6
    executor: python
    environment:
      TESTS: unit
    working_directory: ~/tests
```


This would resolve to:
```
jobs:
 build:
   steps: []
   docker:
     - image: python:2.7.15    # From job
     - image: postgres:9.6     # From job
   environment:                # Merged:
     ENV: ci                     # From executor
     TESTS: unit                 # From job
   shell: /bin/bash            # From executor
   working_directory: ~/tests  # From job
```

## Using parameters in executors
If you'd like to use parameters in executors, define the parameters under the given executor. Then, when you invoke the executor pass the keys of the parameters as a map of keys under the `executor:` declaration, each of which has the value of the parameter that you would like to pass in.

Parameters in executors can be of the type `string` or `boolean`. Default values can be provided with the optional `default` key.

**Example build configuration using a parameterized executor**
```
version: 2

executors:
  python:
    parameters:
      tag:
        type: string
        default: latest
      myspecialvar:
        type: string
    docker:
      - image: circleci/python:<< parameters.tag >>
    environment:
      MYPRECIOUS: << parameters.myspecialvar >>

jobs:
  build:
    executor:
      name: python
      tag: "2.7"
      myspecialvar: "myspecialvalue"  
```

**The above would resolve to:**
```
version: 2
jobs:
  build:
    steps: []
    docker:
      - image: circleci/python:2.7
    environment:
      MYPRECIOUS: "myspecialvalue"
```
