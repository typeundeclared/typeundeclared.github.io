---
layout: post
title: Love and Concern for Ada
tags: software
---
*Note: This post is getting submitted, even though its a little rougher than I'd really like.  I've got too many projects on the side, and this just isn't quite getting the time it deserves to be great, so its going to have to be just good as-is.*

Ada generally doesn't make the list of popular languages.  It tends to be regarded as an old, archaic language, and a holdover from Department of Defense mandates back in the 1980s and 1990s.  Having said that... I rather love the language.  It has its share of issues, but for reliable embedded software that needs to be maintainable for a long time (more than a decade), I think its the best choice of the main options (Ada, C, C++).

## History of Ada
How old is Ada?  The original draft of Ada, designed by committee and the winner out of four responses to the original DoD RFP, dates to 1979, standardized in 1983. 

Now, 1983 is old.  Older than me, even (heh).  But since I find the age argument of languages sometimes... dated, let's just for comparison bring out the birth-date of C: 1972.  And C++?  1983, essentially the same age as Ada.

Versions of Ada:

* 1983: The original.  Earned a reputation at the time for being slow, cumbersome, and buggy, due to the poor implementation of the tools.
* 1995: The defacto version.  Most people who have written embedded Ada in the last two decades have probably used this version.  Its the one most supported by vendors (Adacore, Greenhills, ~~IBM/Rational~~ Atego Apex, etc).  Added a lot of nice language features, but major updates were made for real-time and concurrency.
* (2004): Not a "real" version per-say, but 2004 was when the Ravenscar Profile was published.  This defined a set of restrictions to the tasking model in Ada programs for predictable behavior and execution in a hard real-time environment.  Not all Ada 95 compilers support this profile.
* 2005: More updates, mostly for objected oriented programming (including syntax sugar), as well as a standard library supporting collections (data structures).
* 2012: Movement towards formal verification, with integrated pre- and post-conditions, as well as expression functions and other syntactical updates.

The following is a good chart, courtesy of Adacore:
[Ada Version Comparison Chart](http://www.adacore.com/adaanswers/about/ada-comparison-chart) 

There is also a companion language of sorts, named SPARK, which is a restricted form of Ada designed for formal verification of the software.  The original versions embedded the contracts in specially formatted comments, but the 2014 version of SPARK merged syntax with the Ada 2012 pre- and post-condition pragma updates, so the contracts are no longer comments but actual source code.  In concept, SPARK sounds really spiffy, but I've not actually used in it any production code so I can't really comment as to its efficacy in terms of writing and maintaining high-reliability code, relative to having good development processes, reviews, and testing already in place.

Other interesting reads:

* [C/C++ and Ada Language Comparison (Ada-centric)](http://www.adahome.com/Ammo/cpp2ada.html)

## The Good
### Readability and Maintenance
I'll right.  I'll admit it.  I'm a maintenance programmer---but aren't we all?  The primary code-base I work on is well-established (read, more than a decade old), but one which also changes with the times.  It started with a monolithic application that ran bare-bones on hardware (no OS), and has evolved to meet modern standards which are taking cues from web application frameworks, application portability, and other sources.

Why do I mention this?  I spend most of my day reading code.  Whether that is tracing through code tracking down a bug, or tracing through the places that need to be changed to interface with new functionality, I read much more code than I write.  To that end, I *really* hope the code I'm perusing is clear, well-written, and readable.

I'm going to say something very opinionated, and I'm not sure I'm articulate enough to argue it enough for most people.  I think Ada is one of the more readable languages out there for several reasons, primarily because it reads and speaks a lot more like natural language:

- Infrequent symbol usage
    - None of the syntax really uses **[]** **{}** or **!**
    - Many fewer places where **()** is necessary
- Operators use less esoteric syntax
    - Equals is **=** and assignment is **:=** (this makes it much harder to actually forget/add an **=** as in C and still be legal)
    - **/=** instead of **!=** (more like how you might write it down on paper)
    - Math operators like modulus and remainder are short-handed as **mod** and **rem**, rather than **%** (do you remember which one **%** is?)
    
In some cases, Ada is significantly more verbose: Object'Address instead of &Object.  But really, when you switch between languages often, which is more explicit (and again, reads and speaks like natural language)?

In other cases, the verbosity is a wash, but the readability is significantly better.  Which is easier to read:

    for (int i = 0; i < arr_length; i++)
    {
        arr[i] = something();
    }

or

    for Index in Arr'Range loop
    
        Arr(Index) := Something;
        
    end loop;

It's also a tad higher-level than C: instead of managing the index myself, I'm telling the compiler, just loop me over all the indices in the array, thanks.

Finally, I think there's something about the language that encourages better naming conventions, variables, types, arguments, etc.  *Of course, that could simply be the company and department I work for: there's no stigma against writing a finding in a peer review to rename variables better, etc... which is wonderful.*

In retrospect, some of the above arguments aren't too far off-base from PEP20 in Python, the [Zen of Python](http://www.ada-auth.org/standards/rationale12.html).

### Real Enumerations and Array Ranges
Unlike C, Ada has actual, strongly-typed enumerations.  You *cannot* assign any generic integer to an instance of an enumeration, nor can you silently convert an enumeration to an integer (though you can explicitly convert using 'Val and 'Pos).

Additionally, arrays in Ada aren't restricted to integer indices which start at 0.  Instead, any discrete type (such as integer or enumeration) can suffice as indices.  This means my array bounds could be from -5 .. 5, or the days of the week.

And while all array instantiations must be constrained, I can declare an array type that is unconstrained (for instance, indices are over the positive numbers, and when I declare an instance of that array, I can define the bounds).

### Decent Language-Level Multitasking Constructs
Once I was introduced to Ada years ago, I think I was fascinated that this was a language that had made multi-tasking a true first-class citizen of the language.  It wasn't a bolt-on library to the language, it was integral.

Why is this important?  It starts to let you think about what you are doing with your threads, and how they are interacting, rather than managing the low-level synchronization details.

- Built in monitors (in the form of protected bodies).  Since really, did you honestly remember to unlock that mutex in the exception handler in every procedure that performs a lock/unlock?
- Sure, its easy to *write* code that uses a bunch of library calls for mutexes, semaphores, condition variables, etc, but its generally much harder to debug and know you've gotten the code right.  When I can use a few keywords to define a task (thread) or protected body (monitor) and visibly see the lexical bounds, then I can more easily concentrate on the thread-thread interactions, rather than worrying about the little low-level details of those library calls.
- Tasks (threads) are known at compile time.  While some from the Erlang camp would count this as a very bad thing, when you are writing high-reliability embedded software, I usually want to know my tasks from the get-go.  I know their priority, and their stack sizes.  There's no question about whether I'll get them or not, or something will go wrong with creating them.
    - This isn't to say that in a non-embedded environment, I can't create and destroy tasks dynamically in Ada.  You can.  In my line of work, though, you don't.

### Strong Static Checking by Compiler
Alright, I'll admit failure here.  I was originally going to try to write up a section on Ada being strongly typed, and C/C++ being weaker typed, with points *A*, *B*, and *C*.  Well... after doing a bunch of reading, I ended up convincing myself that the typing models themselves aren't too far off.  Sure C/C++ does let you implicitly change types in more cases, but not in ways that are overly dangerous.

Why then do I still feel that the compiler helps so much more with Ada projects?  In my own inadequate way of thinking about it, the compiler makes far fewer assumptions for you.  I'll try to explain with a 'C' gotcha that set me back hours a few months back.

I was porting a piece of code from one platform to another, and someone was using some standard C call (don't recall which at this time) but hadn't included the system header file for it.  At the time, GCC was implicitly creating a declaration at the point of usage, and linking it up later.  This ended up working at the time, because GCC guessed correctly based on the parameters.  However, on the new system, the definition of the function used a parameter with a different size (say, a 64 bit int rather than a 32 bit), but the value in the implicitly declared call was a 32-bit value.  So when GCC implicitly declared it, it later ended up linking against a library call with differently sized parameters, and the stack got trashed.  *(Yes, you can check for this with -Wimplicit, but that wasn't turned on for some reason at the time).*

Would you ever have gotten that far in Ada?  Absolutely not.

### Strongly Written Standard and Rationale
Have you ever read a language standard?  I mean, really read one.  (Okay, Okay.  I've read only Ada's... and that was for work.  Sorry).

I may be a nerd, but Ada's language standard seems pretty well written.  It's reasonably clear and precise, carefully stipulates what is optional and mandatory behavior.  What really puts the icing on the cake, though, are the *Rationale for Ada XX* by John Barnes documents that accompany the major version changes.  He breaks down the changes, why they were adopted, caveats, etc, clearly and even a little humorously.

See:

- [Rationale for Ada 2012](http://www.ada-auth.org/standards/rationale12.html)
- [Rationale for Ada 2005](http://www.adaic.org/resources/add_content/standards/05rat/Rationale05.pdf)

## The Bad
### Platform Support and Cost
So you remember how I mentioned above, that most Ada programmers are probably familiar with Ada 95?  Well, one of the reasons for that is that very few vendors actually support anything beyond Ada 95.  A few, like Greenhills, have marginal support for portions of Ada 2005, and Atego (which acquired Rational/IBM Apex) currently lists 2005 support though I don't know how complete it is, but only Adacore fully supports Ada 2005 and Ada 2012.

That... should probably be concerning to most Ada programmers.  Not because Adacore doesn't do a good job---I feel they do a stand-up job, both in a professional sense (their runtime is FAA certified on many real-time platforms), and also the regular release of solid open source versions of their software.  But, with the general lack of other vendors supporting versions beyond Ada 95, Ada developers are left in a difficult position: it appears that only one vendor is actively putting money into their Ada suite.

Furthermore, because Ada is *such* a complex language, porting it to a new platform isn't always the most trivial of matters.  The Adacore runtime is reasonable portable, but new platforms aren't effortless (read: you'll likely pay a vendor to port to a new platform, on top of existing licensing costs).  And if you working efforts requiring DO-178B/C certification, certifying the runtime will cost you, too.

For comparison, nearly every platform out there, even embedded, has at least one C compiler that it comes with out of the box (because they needed it to build parts of the platform itself).  Granted, its C, but ease of migration is certainly a strong business argument for C over Ada.

### Flexibility (on Runtime Checking)
I'm a firm believer that your programming language should never surprise you.  Different Ada compilers/vendors provide significantly different options regarding code generation and runtime checking, both in capabilities and in tuning.  In an embedded field, the ability to turn some runtime checking off/on makes some sense, at least historically: you are embedded, after all, and you may not have the performance to spare for all kinds of runtime checking.  Sure.

But for some reason, more often than I'd like, I find myself investigating a crash/lockup/*bug* where I'm pretty sure I've tracked down the piece of code at fault, I'm staring at it... and I sit there wonderful why I didn't just get a CONSTRAINT_ERROR, rather than whatever erroneous behavior I did receive.

For some reason, after years of working in Ada, I feel like I should be able to read a piece of code, and with a given input, follow the logic and think, '*I should get a constraint error... there*.'  But:

* vendor A might not support that (maybe optional, maybe not) runtime check, or
* it might be turned off by default on vendor B, or
* it is on by default on vendor C but we inherited some build files from five years ago where someone made the choice to turn off some runtime checking options.

To some extent, this dovetails in with the **Platform Support and Cost** section, because all those runtime checks, as well as tunability, that the language supports, cost the vendor in terms of complexity of implementation.

### Sizes of Types vs Objects
*I'm including this because its really a pet peeve of mine, and a subtle spot where things may not work very intuitively.*

This dovetails with the high degree of flexibility in the language.  For instance, if I have an enumeration type, such as:

    type Test_T is (A, B, C, D);
   
then a check of Test_T'Size reveals 2, whereas an instance of Test_T'Size reveals 8.

Why?  Because the compiler is still allowed, for performance and alignment reasons, allowed to expand the size of instances of a type, adding in padding, etc.  This ordinarily wouldn't be bad, except for the surprise factor (and your programming language should never surprise you, *should it?*)

Even if one were to do the following:

    for Test_T'Size use 2;
   
an instance still has a size of 8.  This can really haunt you sometimes if you're trying to determine the size of something to send over the wire, and the type and object sizes are different.

See: ['Size Explanation](https://en.wikibooks.org/wiki/Ada_Programming/Attributes/'Size)