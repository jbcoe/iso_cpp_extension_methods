# Uniform calling syntax lite: extension methods for C++

_Jonathan Coe \<jbcoe@me.com\>_

_Roger Orr \<rogero@howzatt.demon.co.uk\>_

# Motivation

When writing generic code, we are forced to differentiate between calling
member functions and free functions: `x.foo()` and `foo(x)` are not the same
thing. When using concepts to constrain generic code we will run into the same
issue and will have to pick member functions or free functions to describe the
concept. This is undesirable and has led to the standard libary introducing
free function equivalents for functions like `begin`, `end` and `data`.

There have been proposals [REF] suggesting changes to C++'s function resolution
rules to allow free functions to invoke members and vice-versa. We are
concerned about the impact such changes will have on existing code and propose
an opt-in alternative by introducing extension methods and using
concept-constrained free functions.

# Emulating protocols with concepts

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
## Proposed syntax

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
    
    void foo(this const FreeFooable& f)
    {
      foo(f);      
    }
    
    b.foo(); // calls foo(b)

# Required changes

# Comparison with uniform call proposals

# References

