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

  - Represent the diff of a state i.e a structure that represent what is need to
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

        atom.Set("name","bob")

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

        // A more efficient diff structure is:
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

## The Problem
  This current idea is indeed interesting but we have one huge problem, `Structs`.
  Yes the cute, composable buddy of all go programmers is the beast of reactivity,
  one a simple case, we can bend it into submission but because structs can contain
  composed structs or defined interfaces, lets just say you are in for a headache.

  **The Question**
   1. Should you stick to use map like structures for key:value pairs?
   2. Should we allow struct types because we can never escape them and if we
   have certain rules that restricts what can be done in a reactive sense and
   use a wrap structure that safe-guards those rules, will it still be a cool API?

   3. What does a feasible reactive struct look like?

   ```go

    type Address struct{
      Postal string
      Zip string
    }

    type Profile struct{
      Name string
      Age int
      Active bool
      Address *Address
    }

    atom := ReactiveStructs(&Profile{
      Name: "bob",
      Age: 30,
      Active: true,
      Address : &Addres{
        Postal: "43L NY",
        Zip:"11232",
      },
    })

    atom.Signal(Signal{
        "addr": ".address",
        "postal": "543GL LU",
    })

    nameReactorValue := atom.Get("Name") // => returns a ReactiveValue

    nameReactorValue.Value() => "bob"
    nameReactorValue.Set("john") // creates a diff structure with the state transition
    nameReactorValue.Get() => "john" // computes the states to return the value 'john'

    addressReactor := atom.Get("Address") //=> returns a ReactiveStruct

    postalReactorValue := addressReactor.Get("Postal") // => ReactiveValue
   ```

## Interesting Ideas


### Redux
  Redux is a retake on the flux framework and borrows from log-based application design where the log is the central source of all truths. That is all changes happen in the log and all reads are taking from the log. I find these design principle to be very attractive, has it simplifies a lot of the complexity of dealing with individual sources of truths. An example of redux code as below:

  ```js

     state = 0
     // a reducer
     cumm = function(state, action){
       switch action {
         case "INCREMENT":
           return state + 1
         case "DECREMENT":
           return state - 1
       }
       return state
     }

     //a central signal store
     store = redux.CreateStore(cumm)

     store.OnChange(function(newstate, oldstate){
       // we get the old state and newstate as notifications of change
     })

     store.Signal({ action: "INCREMENT"})
     store.Signal({ action: "DECREMENT"})


  ```


  Its a rather simple but interesting concept, because all changes take place in the store and others give their intent to create a mutation, rather than mutate the state directly, we encapsulate alot of the complexity of mutations randomly happening and remove all race-conditions that can happen because all reads and writes are done in order rather than out of order,even in concurrent systems, you get this safety with such a design.

## Questions

  1. How can we take this principle from log-based design and redux ideas of reducers(functions that effect the change in stores) into a more flexible go idiomatic API for our reactive library.

   A: We want to have a similar idea for each reactive structure. It should be the cental source of truth. That is we need to squiz out all the fragmentation from the api and ensure all modifications happen with the parent source, that is if their are any deep alias/structures pointing to any sub-level structure, then any write to such a structure must be passed up to the parent, there should be no modification which the parent or source does not know about. The source must perform its reads and writes in a orderly manner (possibly batching reads and writes) thereby limiting any unexpected conditions taking effect.

   - What this solves?
     1. Removes all possible race-conditions within the structure.
     2. Provides a predicable mutation/transformation of any structure which allows easy features like:
       - Time travel
       - Recording/Replay of change/action
     3. It Simplifies our lives.

   2. Do we see some advantage to the ideas of reducers(functions that affect change) for stores?

### Cycle.js
  Cyclejs takes a different but interesting approach in how it handles reactivity by thinking of cyclical feeding systems, where one part is considered the 'Computer' and the other the 'Human'. Each parts consistently and continously feed each other, where the output from one becomes the input of the other and vise-versa.
  At first, It may seem rather odd but its rather a useful approach, allow a natural flow since in truth, every thing we do follow such a principle, eg. The inputs from the human hand to the mouse feeds the cursor on screen which feeds the visual senses of the human which feeds the cursor's direction on screen, a complete cycle of request-response.

  ```js

      function Main(requests){
        dom = requests.DOM
        // response
        return {
          DOM: dom.find('.load').map(function(val){
            return h.Div(h.Ul(h.li(val.getAttribute('id'))))  
          }),
        }
      }

      dom = require('domdriver')
      drivers = {
        DOM: dom,
        Ajax: ...,
      }

      cycle.Run(Main,drivers)
  ```

  Like the sample code above:
   Cycle has a main Run(Main,Drivers) method that handles the feeding of drivers, that generate requests which are passed into a Main() function, that listen for those requests and replies with responses(using the namespace for the target driver). It heavily relies on Rjx(ReactiveX) library to provide the continuous, reactive stream of inputs and outputs. By handling the feed loop and cycle of outputs response into input requests,it provides the cycle of 'Computer -> <- Human' interaction. A very powerful constructs.

   Questions:
   1. What exists in Cyclejs, that we can benefit from?
   2. What part of these structure provides something interesting for us?
   3. Do we need or adopt some piece or part of its idea?

## Conclusion:
  I believe all these ideas portrayed here are interesting but are not necessarily the solution. I believe we can not achieve a total reactive library that utterly simplifies all use cases in go but rather we can adopted specific patterns like Redux and Cyclejs, which both simplifies our codes and provide the needed reactivity we want.

  We have been trying to create a library that embodies a total construct for reactivity but this will end up to be rather a large, bloated cases of reflect packages and its mix. Go is a very unique language and in truth, when you fill like you are doing something 'Clever' then Stop, Think and Reverse to something simple.

  I think adopting the ideas in here into patterns and yes, repetitive patterns we can use in code, as per needed by the situation is a more useful approach.

  The Key is the Patterns and not a Library.
