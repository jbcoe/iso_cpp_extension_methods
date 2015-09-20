# Uniform-calling-syntax lite: extension methods for C++

_Jonathan Coe \<jbcoe@me.com\>_

_Roger Orr \<rogero@howzatt.demon.co.uk\>_

#1. Motivation
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

# Constraining free-functions with concepts
Without language changes we can introduce free functions that will be applied
only to objects which support member function(s) with a given name and signature.

We can define protocol-like concepts that will match any type with a given
signature.  This has the advantage over Swift's protocols (and Java's
interfaces) that user-defined types do not need to know about the concept to
adhere to it. Swift's protocols require classes to explicitly declare that they
implement them.

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
Extension methods are free functions that can be invoked with member-function
syntax. Adding extension methods to c++ requires a language change. Addition
of extension methods was previous proposed [REF]. We believe that the impending
introduction of concepts into C++ makes them a vastly more appealing
proposition.

    struct B;
    
    // 'this' makes 'foo' an extension method   
    void bar(B* this, int x);

    B b;
    b.bar(); // invokes bar(&b)

The first argument of an extension method is named `this`, further arguments
are passed to the function when it is invoked as a member function.  Extension
methods do not need to be invoked as member functions but can be invoked as
free functions.

We can write generic extension methods and constrain them with concepts.  We
can use this to create a member function equivalent only when a given free
function invocation is valid.

    class C;
 
    void foo(C& c, int x);

    C c;
    C.foo(0); // will not compile

    template <typename T>
    concept bool FreeFooable
    {
      return requires(T& t, int i)
      {
        {foo(t, i)} -> void;
      };
    }
    
    // 'this' makes 'foo' an extension method   
    void foo(FreeFooable* this) 
    {
      foo(*this);      
    }
    
    c.foo(); // calls foo(c)

The behaviour of extension methods below is designed to minimise the impact on
existing well-formed code (Ignoring SFINAE tricks).  

##Overload resolution
The decision to invoke an extension method is made at compile time as part of
overload resolution.  If a member function could be invoked with the supplied
arguments then the member function will be chosen in preference. Extension
methods should only be invoked when there is no possible member function
invocation.

    struct D
    {
      void foo(unsigned u);
    };

    void foo(D* this, int i)
    {
      // some implementation
    }

    D d;

    d.foo(3); // invokes D::foo(3);

Adding the extension method does not break existing code, but the behaviour may
be undesirable. Compiler warnings may be desirable if an extension method has
entered an overload set but is not selected.

Non-public or deleted member functions can prevent an extension method from
being invoked as they will be preferentially selected during overload
resolution but rendered non-invokeable. 

##Access to class members
Extension methods are like normal free functions apart from their invocation
syntax and have no access to non-public members of a class.

    struct E
    {
      private:
      void foo();
    };

    void bar(E* this)
    {
      this->foo(); // will not compile as E::foo() is private
    }

##Interaction with virtual functions
Extension methods do not interact with a virtual functions. If a derived class
has a virtual method with the same signature as an extension method that is not
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
    bw.wobble(1.0); // invokes extension method wobble(auto* this, double d)

The non-invocation of derived class member functions may be non-desirable but it cannot
break code. Adding an extension method cannot change the behaviour of compiling code.
Adding a member function to a class where extension methods are defined with the same
signature may result in non-obvious behaviour. Compiler warnings may be desirable.
[Note that the issue above is already present when derived class methods
shadow virtual base class methods.]

# Comparison with existing uniform call proposals
We had to write some boilerplate (maybe more language changes could eliminate
or simplify this) but have something resembling uniform call syntax. 

Extension methods are required to have `this` as the first argument. Other universal
function call syntax papers have considered allowing the object to take any argument position
in a free function when invoked as a member function:

We have made no changes to exisiting overload resolution rules that could cause existing
valid code to fail to compile or to change its meaning. The addition of concept-constrained
free functions or extension methods into existing valid code will not cause it to fail to compile
or change its meaning (ignoring SFINAE-based tricks).

# Open questions and bikeshedding
Should `this` in extension methods be a reference or a pointer? It really should
not be null, so a reference is appealing. In all current contexts where `this`
appears in C++, it is a pointer. The decision is to value consistency or
correctness. The author(s) have selected consistency but this has no deep
implications on the proposed changes.

# References

