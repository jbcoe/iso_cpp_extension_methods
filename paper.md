# Uniform calling syntax lite: extension methods for C++

_Jonathan Coe \<jbcoe@me.com\>_

_Roger Orr \<rogero@howzatt.demon.co.uk\>_

_YOUR_NAME_HERE \<anyone@bsi\>_

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

# Constraining free-functions with concepts
Without language changes we can introduce free functions that will be applied
only to object which support a member function with a given name and signature.

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

# Extension methods
With language changes, we can introduce free functions that can be invoked 
using member function syntax. 

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
    
    void foo(const FreeFooable& f=this) // 'this' makes 'foo' an extension method
    {
      foo(f);      
    }
    
    b.foo(); // calls foo(b)

# Behaviour of extension methods
Member function will be chosen in preference. 
Can shadow similarly named functions in derived classes (diagnostic?).
Can only access public members. 
No interaction with vtable.  

# Required changes
Language changes needed to change function resolution rules (add a bit to allow
extension methods).

# Comparison with uniform call proposals

We had to write some boilerplate (maybe more language changes could eliminate
or simplify this) but have something resembling uniform call syntax.  Changes
in this proposal won't break existing code and when the free functions and
extension methods are declared along with the generic code they will be used
by, allow an opt-in uniform calling syntax.

# References

