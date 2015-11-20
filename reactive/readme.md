# Reactivity in Go
I have being thinking alot about what reactivity in go means, what type of API
fits and What would make for a simple but elegant, approach to how this api will be
used. Go is unlike most languages and the construct we use, those beautifully design
do bring their own set of challenges.
I think in order to truly bring in a level of reactivity whilst still maintaining
a go idiomatic approach, we need to resolve the following problems:

  - How do you bring reactivity with structs.
  - How do you manage reactivity with pointer structures (maps and arrays/slices)
  - How do you manage mutation.

That is in simple terms, reactivity and immutable state are absolutely in the same box
Reactivity denotes change, change of state and by understanding state as immutable and
a change as a occurence of a new state rather than a change in state, we can build very
flexible but simple systems unlike if we tried managing the states as mutable and try
to manage the transitions of change by such a state.

The real keys and usefulness of any reactive system is based on:

  1. The ability to define a immutable
  2. The ability to track changes of such a state over a set period of time(t).

      ```
        Reactive(U)----->Mutation over Time(t)----------CurrentReactive(U)
         |                                            |
         |---R1----R2----------------------Rt---------|
      ```
  3. The ability to transmit those changes in a simple stack system, where this
  changes propagate down to any consumer

  4. The ability to apply a computation to create a new state.

## The API Vision
 Our vision of what a go reactive API must be is:

  1. Simple
  2. Immutable Structures
  3. Observable
  4. Computable

 I think the core to attain all this lies in how the basic structure of the language
 can be turned into observable and immutable structures and understanding how these
 structures represent change and are able to reconcile such changes (i.e to transform into another state)

 First Key: **The reactive structures must be diffable**
 Diffable in the sense we can represent what causes the change from a current state
 to the next. I think a log like append-only structure is suited for this type of thinking.
 Just as a log shows the transition of a series of operation from one point to the next, we
 need a structure that lets us map not the final representation of a state but rather the minimum  
 computation needed to move from the initial state to the desired one.

 For basic types (string,bool,rune) this is simple the transition of one base
 value to the next, e.g

  ```go
    count := 1
      |--------2--------------3----------4--------5--------n
  ```

 For higher level basic types (Arrays,Slices,Map) there needs to be a system in
 place for each type that, allows the following:

  1. Represent the diff of a state i.e a structure that represent what is need to
   move from one state to the next e.g

     Using our DREAM API:
     ```go

        atom := ReactiveMap(map[string]interface{}{
          name: "john",
        })

        // Listen will return the atom itself because we should not necessary care
        // about 'what' changes but rather about 'who' changes, when we so much care
        // about 'what' we enter a tangled mess trying to keep track of minute details.
        // We want global details, 'who' changed then we can take 'who' and get its 'what's
        // and do whatever we want with them.
        atom.Listen(func (atom){})

        atom.diff() => {}

        atom.ToMap() => {name: "john"}

        atom.Set("name","john")

        diff(atom) => {name:"bob"}

        atom.ToMap() => {name: "bob"}

        atom.Set("age",21)

        diff(atom) => {age:21}

        atom.ToMap() => {name: "bob",age: 21}

        atom.Set("name","alex")

        diff(atom) => {name:"alex"}

        atom.ToMap() => {name: "alex",age: 21}

        atom.Remove("name")

        diff(atom) => {name:"alex"}

        atom.ToMap() => {age: 21}

        // A total diff map of the structure
        AllDiff(atom) => [{name:"john"},
        {Action: ADD, props: {"name":"bob"}},
        {Action: ADD, props: {"age":21}},
        {Action: ADD, props: {"name":"alex"}}
        {Action: REMOVE, props: {"name":"alex"}}]

        // A more efficient repsentation is:
        AllDiff(atom) => {
         "name": ({val: "john"}, next: {val:"bob", next: {"alex", next: {val:"", next: nil}}}),
         "age": ({val: nil}, next: {val:21, next: nil}),
       }
     ```

     What do we see in the structure above?

     1. We see a reactive map that is immutable and changes

     2. We see a series of changes applied to get a new state.

     3. We see a diff'ed structure that does not show us ***the final state***
     but the **computation** that is required to get to that state.

   That is our dream api should represent the series of computation to attain
   a final state and not this final state itself. When ever you get a value
   from such a structure,it should compute the state for that field.
