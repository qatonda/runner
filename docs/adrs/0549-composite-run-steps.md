TODO: Change file name to represent the correct PR number (PR is not created yet for this ADR)

# ADR 0549: Composite Run Steps

**Date**: 2020-06-17

**Status**: Proposed

## Context
Customers want to be able to compose actions from actions (ex: https://github.com/actions/runner/issues/438)

An important step towards meeting this goal is to build in functionality for actions where users can simply execute any number of steps. 

## Decision
In this ADR, we support running multiple steps in an Action. In doing so, we build in support for mapping and flowing the inputs, outputs, and env variables (ex: All nested steps should have access to its parents' input variables and nested steps can overwrite the input variables).

### Steps
Example `user/test/composite-action.yml`
```
...
using: 'composite' 
steps:
  - run: echo hello world 1
  - run: echo hello world 2
```
Example `workflow.yml`
```
...
jobs:
  build:
    runs-on: self-hosted
    steps:
    - uses: actions/checkout@v2
    - uses: user/test@v1
    - name: workflow step 1
      run: echo hello world 3
    - name: workflow step 2
      run: echo hello world 4
```
Example Output
```
echo hello world 1
echo hello world 2
echo hello world 3
echo hello world 4
```
We add a token called "composite" which allows our Runner code to process composite actions. By invoking "using: composite", our Runner code then processes the "steps" attribute, converts this template code to a list of steps, and finally runs each run step 
sequentially. 

### Inputs
Example `user/test/composite-action.yml`:
```
using: 'composite' 
inputs:
  your_name:
    description: 'Your name'
    default: 'Ethan'
steps: 
  - run: echo hello ${{ inputs.your_name }}
```

Example `workflow.yml`:
```
...
steps: 
  - id: foo
    uses: user/test@v1
    with:
      your_name: "Octocat"
  - run: echo hello ${{ steps.foo.inputs.your_name }} 2
```

Example Output:
```
hello Octocat
Error
```

Each input variable in the composite action is only viewable in its own scope (unlike environment variables). As seen in the last line in the example output, in the workflow file, it will not have access to the action's `inputs` attribute.

### Outputs
Example `user/test/composite-action.yml`:
```
using: 'composite' 
inputs:
  your_name:
    description: 'Your name'
    default: 'Ethan'
outputs:
  bar: ${{ steps.my-step.my-output}}
steps: 
  - id: my-step
    run: |
      echo ::set-output name=my-output::my-value
      echo hello ${{ inputs.your_name }} 
```

Example `workflow.yml`:
```
...
steps: 
  - id: foo
    uses: user/test@v1
    with:
      your_name: "Octocat"
  - run: echo oh ${{ steps.foo.outputs.bar }} 
```

Example Output:
```
hello Octocat
oh hello Octocat
```
Each of the output variables from the composite action is viewable from the workflow file that uses the composite action. In other words, every child action output(s) is viewable only by its immediete parent.

Moreover, the output ids are only accessible within the scope where it was defined. In the example above, note that in our `workflow.yml` file, it should not have access to output id (i.e. `my-output`). For example, in the `workflow.yml` file, you can't run `foo.steps.my-step.my-output`.

### Context
Similar to the workflow file, the composite action has access to the [same context objects](https://help.github.com/en/actions/reference/context-and-expression-syntax-for-github-actions#contexts) (ex: `github`, `env`, `strategy`). 

### Environment
Example `user/test/composite-action.yml`:
```
using: 'composite' 
env:
  NAME2: test2
  SERVER: development
steps: 
  - id: my-step
    run: |
      echo NAME1: ${{ env.NAME1 }} 
      echo NAME2: ${{ env.NAME2 }} 
      echo Server: ${{ env.SERVER }} 
```

Example `workflow.yml`:
```
env:
  NAME1: test1
  SERVER: production
steps: 
  - id: foo
    uses: user/test@v1
  - run: Server: ${{ env.SERVER }} 
```

Example Output:
```
NAME1: test1
NAME2: test2
Server: development
server: production
```

We plan to use environment variables for Composite Actions similar to the parent/child relationship between nested function calls in programming languages like Python in terms of [lexical scoping](https://inst.eecs.berkeley.edu/~cs61a/fa19/assets/slides/29-Tail_Calls_full.pdf). In Python, let's say you have `functionA` that has local variables called `a` and `b` in this function frame. Let's say we have a `functionB` whose parent frame is `functionA` and has local variable `a` (aka `functionB` is called and defined in `functionA`). `functionB` will have access to its parent input variables that are not overwritten in the local scope (`a`) as well as its own local variable `b`. [Visual Example](http://www.pythontutor.com/visualize.html#code=def%20functionA%28%29%3A%0A%20%20%20%20a%20%3D%201%0A%20%20%20%20b%20%3D%202%0A%20%20%20%20def%20functionB%28%29%3A%0A%20%20%20%20%20%20%20%20b%20%3D%203%0A%20%20%20%20%20%20%20%20print%28%22a%22,%20a%29%0A%20%20%20%20%20%20%20%20print%28%22b%22,%20b%29%0A%20%20%20%20%20%20%20%20return%20b%0A%20%20%20%20return%20functionB%28%29%0A%0A%0A%0AfunctionA%28%29&cumulative=false&curInstr=14&heapPrimitives=nevernest&mode=display&origin=opt-frontend.js&py=3&rawInputLstJSON=%5B%5D&textReferences=false) 

Similar to the above logic, the environment variables will flow from the parent node to its children node. More concretely, whatever workflow/action calls a composite action, that composite action has access to whatever environment variables its caller workflow/action has. Note that the composite action can append its own environment variables or overwrite its parent's environment variables. 


### If condition?
TODO: Figure out: What does it mean if the composite step is "always()" but inner step is not? What should the behavior be:
steps:
  - uses: my-composite-action@v1
    if: cancel()
    
### Visualizing Composite Action in the GitHub Actions UI
We want all the composite action's steps to be condensed into the original composite action node. 

Here is a visual represenation of the [first example](#Steps)
```
| composite_action_node |
    | echo hello world 1 | 
    | echo hello world 2 |
| echo hello world 3 | 
| echo hello world 4 |

```


## Conclusion