---
layout: post
title: Typed Racket - Enumerations and other Comments
---

I recently spent a bit of time working through some Racket and a simple web application.  I've worked some in Lisp/Scheme in the past (school, specifically), so at first it was refreshing to get back into that swing.  However, I rather quickly ran into one of my least favorite aspects of interpretted code: I really don't like to find out about silly errors (wrong order of arguments, typos, etc) at runtime.  (This, coincidentally, is my least favorite aspect of Python, though I highly appreciate its clarity and comprehensive libraries).

I was therefore tempted to adapt a portion of my simple web application to Typed Racket; specifically, I switched over the back-end storage and the model, but (for now) left the user interface code as plain Racket.  While I think it helped raise confidence about the code I had written, there were a few things that didn't turn out quite as I expected once I dug into things.  Here, I'll discuss typing and enumerations, along with a few other comments on experiences with Typed Racket.

# Enumerations and Typed Racket

Coming from Ada, I use enumeration types a *lot*, as you can do a lot of nice things with them.  I had an aspect of my code that I thought lent itself well to enumerations: state of a task.  I figured on trying to make something in Racket that operates somewhat similarly.  One might argue I should learn to do things the Lisp/Scheme/Racket way: they're probably right, but I figure this is a good exercise to help me relearn (and be forced to read some to figure out decent ways to do these things).

## Initial Type Declaration and Iteration

In Ada, I'd declare something like the following:

    type Task_State is (backlog, open, working, closed, archived);
    subtype Primary_State is Task_State range open .. closed;

The first thing I'd like to do is iterate over Primary States.  Easy to do in Ada:

    for state in Primary_State loop [...] end loop;

So let's define some Typed Racket.  From what I've read, the right way to do this is (ignoring the subtyping for the moment):

    (define-type State (U 'open 'working 'closed))

Simple enough.  However, I could not figure out how to iterate over those types.  Then I realized that the type system doesn't fully support a truly discrete-_only_ type declaration.  (U .. ) is designed to handle things like:

    (U Number 'nada)
    (U (Pair Number Number) 'nada)

So how would one iterate over that?  You can't.  I could define a State-range list:

    (define-type State (U 'open 'working 'closed))
    (define State-range '(open working closed))

Except that repeats itself.  Time to learn some macros?  After a little playing around and reading some guides, I came up with this first instance.  The hardest part was figuring out how to make a new identifier.  While I did not read the entire write-up, I found the following useful: [Fear of Macros](http://www.greghendershott.com/fear-of-macros).

    (define-syntax (define-enum-type stx)
      (syntax-case stx ()
        [(_ name (val ...))
         (with-syntax ([lname (format-id stx "~a-~a" #'name "range")])
           #'(begin
               (define lname (list val ...))
               (define-type name (U val ...))))]))
    (define-enum-type State ('open 'working 'closed))

Now I have a State-range list that I can iterate over (if I want).

## Lookup Tables
The next thing I want to do is look up the next and previous states, so I can implement some state changes.  I need to be able to do something like saying Working => Closed, but Closed => Closed (no change).  In Ada, I could do:

    Next_State : constant array (Task_State) of Task_State :=
      (Open => Working,
       Working => Closed,
       Closed => Closed);

Now, I know that Next_State will give me an answer for any valid state.  Great.  Time to set this up for Racket:

    (: next-state (-> State State))
    (define (next-state state)
      (case state
        [(open) 'working]
        [(working) 'closed]
        [(closed) 'closed]))

Now, this works.  But I can foresee wanting to do previous-state, as well as integer->state and state->integer.  These are all sorts of pair lookups, so perhaps zipping two lists together as an association list might be nice?  I coded up something with (assoc) but had problems working through the type checker.  Realizing that one can type a function into the typed REPL and get the full type signature (woohoo), I finally figured out (assoc) itself is the problem:

    > assoc
    - : (All (a b c)
          (case->
           (-> Any (Listof (Pairof a b)) (U False (Pairof a b)))
           (-> c (Listof (Pairof a b)) (-> c a Any) (U False (Pairof a b)))))

See it?  (assoc) can return a False.  That makes sense: a general, run-of-the-mill association list might not have the element you're looking for.  But I'm reasonably certain the list I'll be generating will have every State, but without coding up something specific, I don't think I can convince the type checker.  I could just code up something that works around the False part of the union, but decided to see what else was out there.

I tried to work around the issue using a combination of filter and list-accessors:

    (: zip (All (a b) (-> (Listof a) (Listof b) (Listof (List a b)))))
    (define (zip listA listB)
      (map (λ ([x : a] [y : b]) (list x y)) listA listB))

    (: zip-assoc-value (All (a b) (-> a (Listof a) (Listof b) b)))
    (define (zip-assoc-value value listA listB)
      (second (first (filter (λ ([x : (List a b)]) (eq? (first x) value)) (zip listA listB)))))

    (: next-state (-> State State))
    (define (next-state current)
      (zip-assoc-value current State-range '(working closed closed)))

It works, but I feel like I'm subverting the type system a bit?  Technically, I have the same problem as before: I _might not_ actually find the value I'm looking for, but my function's type doesn't specifically state that.  It'll just throw a runtime error.

On other fronts, I've seen references to hashes, which is really just another form of association list, but perhaps I'll play around in that realm.  So what's the type of (hash-ref)?

    > hash-ref
    - : (All (a b c)
          (case->
           (-> (HashTable a b) a b)
           (-> (HashTable a b) a False (U False b))
           (-> (HashTable a b) a (-> c) (U b c))
           (->* (HashTableTop a) (False) Any)
           (-> HashTableTop a (-> c) Any)))

Uhh.  Hrmm.  It's not written up the same way as (assoc), namely, the base case doesn't handle the condition of a key not found.  I believe that means the function just throws an error; in some ways, that's less ideal (in my mind) than just making one deal with the possibly due to the typing system declaring it.  (Yes, yes, I'm throwing stones at glass houses, since I started on this exploration because I wanted an association of some kind that _wouldn't_ make me deal with failed lookups).

Side note: the next several types for hash-ref are all covering whether failure-result is included as an option; according to the docs, HashTableTop is a read-only kind of superset of hash table with unknown types.

A quick check that I'm reasoning decently about using hash-tables for this:
    
    > (define-type State (U 'open 'working 'closed))
    > (: state->integer-table (HashTable State Integer))
    > (define state->integer-table (make-hash '((open . 0) (working . 1) (closed . 2))))
    > state->integer-table
    - : (HashTable State Integer)
    '#hash((closed . 2) (working . 1) (open . 0))

This certainly feels closer to where I want to be, at least in terms of elegance, though it's arguably both better and worse than the association lists

# Comments: Type Annotations

Found out that you have to annotate the for/* functions.  It took a bit of special syntax that took me a while to figure out; once I realized how to read the docs, it made sense, but it wasn't overly obvious.  See: [Docs](http://docs.racket-lang.org/ts-reference/special-forms.html#%28form._%28%28lib._typed-racket%2Fbase-env%2Fprims..rkt%29._for%29%29).  The parantheses especially can be a little tricky when you have multiple bindings (or I'm just rusty in Lisp?  Nah...)

    (for/list : (Listof (Pair State Integer))
        ((x : State State-range)
         (y : Integer (in-range 3)))
        (cons x y))
    - : (Listof (Pairof State Integer))
    '((open . 0) (working . 1) (closed . 2))

# Comments: typed/db

When I first started working on the kanban board, I was using plain racket and the sqlite interface.  Having since started down the path of typed racket, I was assuming at some point I'd ditch the db interface (as there didn't appear to have been any sort of ORM management).  To my surprise, there is a typed/db package, and I played around with it.

The first issue I had some effort working around was that pretty much every query interface returned an SQL-Datum, which is really just an Any.  I haven't figured out (yet) how to coerce types (assuming that's possible/intended), so when fetching query values, I had to wrap the usage in them with conditionals about their types.

    (: kanban-programs (-> kanban (Listof String)))
    (define (kanban-programs a-kanban)
      (filter (λ (x) (string? x))
              (for/list : (Listof SQL-Datum)
                ([p : (Vectorof Any) (query-rows (kanban-db a-kanban)
                                      "select program from tasks group by program order by program")])
                (vector-ref p 0))))

So for instance, query-rows returns (Listof (Vectorof Any)), so I unpack the vector, returning the first element, and then filter out anything that isn't a string.  However, the vector-ref is a bit ugly.  Really, I had originally started this in the non-typed code with (in-query).  _But_... I ran into this:

    > (sequence->list (in-query (kanban-db a-kanban) "select program from tasks group by program order by program"))
    . . in-query: broke its own contract;
     promised a vector,
      produced: "personal"
      in: an element of
          the range of
          (->*
           (any/c any/c)
           (#:fetch
            any/c
            #:group
            any/c
            #:group-mode
            any/c)
           #:rest
           (listof Any)
           (sequence/c (vectorof Any)))
      contract from: (interface for in-query)
      blaming: (interface for in-query)
       (assuming the contract is correct)
      at: <pkgs>/typed-racket-more/typed/db/base.rkt:72.2

This ultimately resulted in: (Return type of in-query wrong for single-column queries? #460)[https://github.com/racket/typed-racket/issues/460]  Punch-line: typed racket cannot (currently) describe the function itself (it returns non-fixed multiple values).  Wasn't hard to switch the usages in my code.

# Conclusions

Ultimtaely, while I did appreciate working in typed racket (a staticly typed Lisp!), and I ultimately have more confidence in the model portion of my code (before any testing), I'm not sure that it added the degree of static checking that I was hoping it would, at least for the effort involved.  I'm more confident that types and the arity of calls all match up now, but I don't quite feel that the typing system has helped me eliminate as many other kinds of runtime errors as other static typing systems.  (To be fair, this is a judgement call, and I've not yet worked with Typed Racket sufficiently, so perhaps further experience will prove me wrong).
