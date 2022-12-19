# Project: Cromwell
To compile SQLite into a dot-net DLL.

SQLite is written in C, delivered as gigantic single C code file that you can drop into any project. While this works well if your code runs in a similar environment to C, there's an "interop" layer when you try to use SQLite as part of a C#/dot-net managed environment.

It work surprisingly well, but that hand-over still represents a bit of overhead. Wouldn't it be nice to get rid of it altogether?

This is a project to compile the SQLite "augmentation" C code into C#. It won't be particularly readable C# code, but it'll compile and run. As C# code, the dot-net JIT compiler can make run-time optimizations and there's no need for an interop hand-over between worlds.

I am grateful to Hacker News user "kevingadd" [for inspring me](https://news.ycombinator.com/item?id=33513842) to have the original idea.

# But Why?

- The dot-net runtime can do a better job when dot-net code hands over to dot-net code without the interop. The JIT layer will have both sides and can make optimizations it couldn't otherwise make.
- Automatic garbage collection is sold as being more efficient than explicit calls to `free` to release memory. As `free` will be implmented as an empty function in the converted code, we shall perhaps see if that is indeed the case.
- C# is a "safe" language. If someone should find a buffer-overflow vulnerability in the C version of SQLite, we're covered.
- I'd like to try writing a compiler. C is a nice simple language to read and C# is a nice simple language to target.
- The work could be refactored to convert SQLite into Python and other similar managed languages instead of C#.
- While I'm restricting myself to SQLite, I hope this core converter code could be put to other libraries written in C. The Linux kernal as a dot-net library anyone?

I'm doing this in public to help get feedback from other developers. As of right now, I'm not sure any of this is actually a good idea. In fact, the remainder of this page is really just the plan that my brain insisted on writing instead of sleeping. 

And there are good reasons to consider this whole thing a waste of time.

- SQLite is pretty damn fast and I barely notice the interop hand-over. Is this a whole lot of work to save me a millisecond?
- SQLite is not known for security vulnerabilities, despite being written in C.
- C is a simple language? What are you going to do if you stumble upon some code that casts a pointer-to-function into an array of bytes? Are you quite sure SQLite doesn't do that?

So if you're reading this because I posted a link on Hacker News or similar, I am please asking you the question, is this project worth going ahead with? Please leave a comment where you found this link.

## The Name

**Project: Cromwell** is named after *Bernard Cromwell*, the author of the *Sharpe* books. (Cromwell adds Sharpe - Just like this project converts C to C-Sharpe. Geddit?)

I can be persuaded to change the name. The author is still alive and this project is really nothing to do with him. Also, "Cromwell" is the surname of a King of England who did some rather unpleasant things. That decision will only come if I do decide to go ahead with developing this project. It would be weird to change the name and then immediately abandon the project.

# The Plan

"If you fail to plan you plan to fail."

## 1. Read the SQLite "augmentation" C code into structured data.

Preprocess. Tokenize. Structure.

At the end of this will be the C source converted into a struture. There'll be an object where if you open it and drill down deep into the contents, you'll find an `if` object with `Condition`, `Then` and `Else` properties. This will need to have awareness of state here, as `name '*' name` might be a multiplication or it might be a declaration of a pointer type.

## 2. Generate C# code.

With that gigantic structure, churn out C# code. For this phase, we're only aiming for simple working code - not neccessarily efficient code. It'll be some time before we run a unit test so I'd rather we had the simplest code, leaving making good code for a later phase.

As an example, here's how the function `sessionSkipRecord` would be converted, with the original C code in comments.

```
/* static void sessionSkipRecord(u8 **ppRec,int nCol) { */
void Conv_sessionSkipRecord(MutableValue_PointerTo_PointerTo_u8 ppRec_Param, MutableValue_int nCol_Param)
{
    var ppRec = CloneParam(ppRec_Param)
    var nCol = CloneParam(nCol_Param);

    /* u8 *aRec = *ppRec; */
    var u8 = Dereference(ppRec);
    
    /* int i; */
    var i = Undefined_int();
    
    /* for(i=0; i<nCol; i++){ */
    var value_0 = FromLiteral_int(0);
    for (Set(i, 0); LessThan(i, nCol), Increment(i))
    {
    
        /* int eType = *aRec++; */
        var aRec_Derefence = Dereference(aRec);
        aRec.Increment();
        var eType = aRec_Dereference;
        
        /* if( eType==SQLITE_TEXT || eType==SQLITE_BLOB ){ */
        var eType_SQLITE_TEXT_IsEqual = IsEqual(eType, SQLITE_TEXT);
        var eType_SQLITE_BLOB_IsEqual = IsEqual(eType, SQLITE_BLOB);
        var eType_SQLITE_TEXT_IsEqual_eType_SQLITE_BLOB_IsEqual_OR = IsOr(eType_SQLITE_TEXT_IsEqual, eType_SQLITE_BLOB_IsEqual);
        if (eType_SQLITE_TEXT_IsEqual_eType_SQLITE_BLOB_IsEqual_OR)
        {
        
            /* int nByte; */
            var nByte = Undefined_int();
            
            /* aRec += sessionVarintGet((u8*)aRec, &nByte); */            
            var nByte_AddressOf = AddressOf(nByte);
            var sessionVarintGet_Result = Conv_sessionVarintGet(aRec, nByte_AddressOf);
            UpdateAdd(aRec, sessionVarintGet_Result);
            
            /* aRec += nByte; */
            UpdateAdd(aRec, nByte);
            
        /* }else if( eType==SQLITE_INTEGER || eType==SQLITE_FLOAT ){ */
        }
        else
        {
            var eType_SQLITE_INTEGER_IsEqual = IsEqual(eType, SQLITE_INTEGER);
            var eType_SQLITE_FLOAT_IsEqual = IsEqual(eType, SQLITE_FLOAT);
            var eType_SSQLITE_INTEGER_IsEqual_eType_SQLITE_FLOAT_IsEqual_OR = IsOr(eType_SQLITE_INTEGER_IsEqual, eType_SQLITE_FLOAT_IsEqual);
            if (eType_SQLITE_INTEGER_IsEqual_eType_SQLITE_FLOAT_IsEqual_OR)
            {
                /* aRec += 8; */
                var value_0 = FromLiteral_int(8);
                UpdateAdd(aRec, value_8);
            }
        }
    }

    /* *ppRec = aRec; */
    var pRec_Dereference = Dereference(pRec);
    Set(pRec_Dereference, aRec);    
}
```

A few things to unpack here. The line `Set(i,0);` appears to be a function that takes a local variable and a value and sets that local variable to the supplied value. If you're thinking that's not how C# works, you'd be right except you don't know what type `i` has. `Undefined_int()` which sets the value of `i` returns type `MutableValue_int`. This is a class with a mutable `int` value inside.

The reason we're storing integer values inside objects is found elsewhere in this function. There's another `MutableValue_int` object named `nByte` and we need to use the `&` "address of" operator on it. For this to work, we need an `AddressOf` C# function that takes a `MutableValue_int` object and returns a `MutableValue_PointerTo_int` object and for that object to return the original `MutableValue_int` object if `Dereference` is called on it. These things have a price.

At the top of this function is a parameter of type `MutableValue_PointerTo_PointerTo_u8`. This has a `Dereference` function that returns a `MutableValue_PointerTo_u8`. All of these `PointerTo` classes have a `Add` and `Increment` functions, just in case there's an array hiding behind the pointer, not just a single value. (Which is why all these `MutableValue_int` objects all contain a `List<int>` with an index inside, just in case there's an `AddressOf-Increment` combo coming up.)

Also, observe the function began by passing the two parameters into the `CloneParam` function. This is here because C allows any function to modify its own parameters, but the caller's copy is unchanged. As we've inadvertantly made all parameters pass-by-reference, this step at the start of every function restores the status quo.

If all this complexity behind the scenes seems like a waste, remember that the objective of this phase is to produce *working* code in as simple a way as possible, because I want to reach the point where unit tests run successfully as soon as possible. A later phase, once we have unit tests, will be to apply optimizations. Could this `MutableValue_int` wrapping a `List<int>` be replaced with a simple `int` because just this one doesn't get address-of'd? Maybe, but that's for later.

## 4. Write library code.

The code generated by the previous phase won't compile because it will be expecting functions like `Undefined_int` and many varieties of `Derefence`. Types like `MutableValue_int` and `MutableValue_PointerTo_int` will be missing and needing writing. (This will also be the point when I decide if `MutableValue<T>` and `MutableValue_PointerTo<T>` would work better, but that's a decision for future-me.)

Also during this phase, the C standard library will need to be written. Functions like `strcpy`, `fopen`, etc will all need to be supplied. `printf` day will be interesting.

## 5. Write an interface.

SQLite's public interface as specified in `sqlite.h` will be converted into C#, but it'll have a horrible interface, expecting `MutableValue_PointerTo_char` values instead of strings. At this phase, we want to wrap those functions with a nice facia that uses sensible types that a C# developer would expect. So it can be a drop-in replacement for interop-C sqlite, the target should be something that uses the same interface as that interop-C class. The code written at this level would pass the parameter values through the neccessary converter, call the underlying functions and finally convert that function's response back into a nice type.

## 6. Convert SQLite's unit tests.

SQLite comes with a comprehansive set of unit tests, so it would be a shame not to use them. Those are written in TCL but there should be a reasonable way to churn through them and produce unit tests that can be run from the Visual Studio Test Explorer. This would be first time that the code is actually run so this might take a while.

If any tests fail, fix the underlying issue before moving on to the next stage.

## 7. Optimize the generated code.

Finally, we can think back to some of the inefficient code we generated during the earlier phase of development. Considerations like:

* This function does not modify its parameter values, so the `CloneParam` call can be dropped.
* This local variable never has `AddressOf` called on it, so it can be replaced with a normal local variable.
* This function parameter is a `PointerTo`, but the pointer is only ever passed into `Dereference`, so change the callers to call `Dereference`.
* There's an AddressOf-Dereference combo (which may have only appeared thanks to the above step). Eliminate the combo.
* There's a `PointerTo` that never gets added to. Replace it with a single-item variety, changing the `AddressOf` calls to `SingleItemAddressOf`.
* If a `MutableValue_int` only ever has `SingleItemAddressOf`, then it doesn't need to be a `MutableValue_int' any more but a `SingleItemMutableValue_int` instead.

At each stage, the code can be regenerated and the unit tests can be re-run to test the replacement was equivalent. 

# Thoughts?

Please open an issue on this project if you feel the plan needs adjusting to make it work better. If you have more general thoughts and comments, you can leave a link where you happened to find the link to this. My website [billpg.com](https://billpg.com/) has lots of social media links if you want to contact me that way.

<div><a href="https://billpg.com/"><img src="https://billpg.com/wp-content/uploads/2021/03/BillAndRobotAtFargo-e1616435505905-150x150.jpg" alt="billpg.com" align="right" border="0" style="border-radius: 25px; box-shadow: 5px 5px 5px grey;" /></a></div>
