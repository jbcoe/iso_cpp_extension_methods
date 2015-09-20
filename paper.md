# Uniform calling syntax lite: extension methods for C++

_Jonathan Coe \<jbcoe@me.com\>_

_Roger Orr \<rogero@howzatt.demon.co.uk\>_

# Motivation
When writing generic code, we are forced to differentiate between calling
member functions and free functions: `x.foo()` and `foo(x)` are not the same
thing. When using concepts to constrain generic code we will run into the
same issue and will have to (arbitrarily) pick member functions or free
functions to describe the concept. This is undesirable and has led to the
standard libary introducing free function equivalents for functions like
`begin`, `end` and `data`.  In all cases the free function invokes the member
function where it is defined.

There have been proposals [REF] suggesting changes to C++'s function resolution
rules to allow free function invocation syntax to invoke members function and
vice-versa. We are concerned about the impact such changes will have on
existing code and propose an opt-in alternative by introducing extension
methods and using concept-constrained free functions.

# Extension methods in C#

# Pythonic free-function in Python

# Protocols in Swift

# Function invocation in Rust

# Constraining free-functions with concepts
Without language changes we can introduce free functions that will be applied
only to objects which support member function(s) with a given name and signature.

We can define protocol-like concepts that will match any type with a given signature.
This has the advantage over Swift's concepts that user-defined types do not need
to know about the concept to adhere to it. Swift's protocols require classes to
explicitly declare that they implement them.

Any user-defined type that implements the member functions required by the concept
can have the concept-constrained generic free-function invoked upon it.

    struct A
    {
      void foo(); 
    };

    A a;
    foo(a); // will not compile

    template <typename T>
    concept bool Fooable
    {
      return requires(T& t)
      {
        {t.foo()} -> void;
      };
    }
    
    void foo(Fooable& t)
    {
      t.foo();
    }
    
    foo(a); // calls a.foo()

# Adding extension methods to C++
Extension methods are free functions that can be invoked with member-function syntax.
Adding extension methods to C++ requires a langauge change.

    class B;
 
    void foo(B& b);

    B b;
    b.foo(); // will not compile

    template <typename T>
    concept bool FreeFooable
    {
      return requires(T& t)
      {
        {foo(t)} -> void;
      };
    }
    
    // 'this' makes 'foo' an extension method   
    void foo(FreeFooable* this) 
    {
      foo(*this);      
    }
    
    b.foo(); // calls foo(b)

# Behaviour of extension methods
The behaviour of extension methods below is designed to minimise the impact on
existing well-formed code (Ignoring SFINAE tricks).  

##Overload resolution
The decision to invoke an extension method is made at compile time as part of
overload resolution.  If a member function could be invoked with the supplied
arguments then the member function will be chosen in preference. Extension
methods should only be invoked when there is no possible member function
invocation.

    struct A
    {
      void foo_A(unsigned u);
    };

    void foo_A(A* this, int i)
    {
      // some implementation
    }

    A a;

    a.foo_A(3); // invokes A::foo_A(3);

Adding the extension method does not break existing code, but the behaviour may
be undesirable. Compiler warnings may be desirable if an extension method has
entered an overload set but is not selected.

Non-public or deleted member functions will prevent the extension method from
being invoked. 

##Access to class members
Extension methods are like normal free functions apart from their invocation
syntax and have no access to non-public members of a class.  

##Interaction with virtual dispatch
Extension methods do not interact with a vtable. If a derived class has a
virtual method with the same signature as an extension method that is not
present in the base class then the extension method will be invoked if accessed
through a base-class reference.

    struct Wibbler
    {
      virtual void wibble(double d);  
    };

    void wobble(auto* this, double d) // unrestricted extension method for brevity
    {
      this->wibble(5*d);
    }

    struct WobblyWibbler : public Wibbler
    {
      virtual void wobble(double d)
      {
        wibble(10*d); // a wobble is a big wibble
      }
    }

    WobblyWibbler ww;
    ww.wobble(1.0); // invokes member WobblyWibbler::wobble(double d)

    Wibbler& bw = ww;
    bw.wobble(); // invokes extension method wobble(auto* this, double d)

The non-invocation of derived class member functions may be non-desirable but it cannot
break code. Adding an extension method cannot change the behaviour of compiling code.
Adding a member function to a class where extension methods are defined with the same
signature may result in non-obvious behaviour. Compiler warnings may be desirable.

# Open questions and bikeshedding
Should `this` in extension methods be a referenc or a pointer? It really should
not be null, so a reference is appealing. In all current contexts where `this`
appears in C++ it is a pointer. The decision is to value consistency or
correctness. The author(s) have no firm opinion.

# Comparison with uniform call proposals
We had to write some boilerplate (maybe more language changes could eliminate
or simplify this) but have something resembling uniform call syntax.  Changes
in this proposal won't break existing code and when the free functions and
extension methods are declared along with the generic code they will be used
by, allow an opt-in uniform calling syntax.

# References

