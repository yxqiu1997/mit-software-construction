# Reading 19: Writing a Program with Abstract Data Types

## Recipes for programming

### **Writing a method:**

1. **Spec.** Write the spec, including the method signature (name, argument types, return types, exceptions), and the precondition and the postcondition as a Javadoc comment.

2. **Test.** Create systematic test cases and put them in a JUnit class so you can run them automatically.

    * a. You may have to go back and change your spec when you start to write test cases. Just the process of writing test cases puts pressure on your spec, because you're thinking about how a client would call the method. So steps 1 and 2 iterate until you've got a better spec and some good test cases.

    * b. Make sure at least some of your tests are *failing* at first. A test suite that passes all tests even when you have'nt implemented the method is not a good test suite for finding bugs.

3. **Implement.** Write the body of the method. You're done when the tests are all green in JUnit.

    * a. Implementing the method puts pressure on both the tests and the specs, and you may find bugs that you have to go back and fix. So finishing the method may require changing the implementation, the tests, and the specs, and bouncing back and forth among them.

### **Writing an abstract data type**

1. **Spec.** Write specs for the operations of the datatype, including method signatures, preconditions, and postconditions.

2. **Test.** Write test cases for the ADT's operations.

    * a. Again, this puts pressure on the spec. You may discover that you need operations you hadn't anticipated, so you'll have to add them to the spec.

3. **Implement.** For an ADT, this part expands to:

    * a. **Choose rep.** Write down the private fields of a class, or the variants of a recursive datatype. Write down the rep invariant and abstraction function as a comment.

    * b. **Assert rep invariants.** Implement a `checkRep()` method that enforces the rep invariant. This is critically important if the rep invariant is nontrivial, because it will catch bugs much earlier. 

    * c. **Implement operations.** Write the method bodes of the operations, making sure to call `checkRep()` in them. You're down when the tests are all green in JUnit.

### **Writing a program** (consisting of ADTs and static methods)

1. **Choose datatypes**. Decide which ones will be mutable and which immutable.

2. **Choose static methods** Write your top-level `main` method and break it down into smaller steps.

3. **Spec.** Spec out the ADTs and methods. Keep the ADT operations simple and few at first. Only add complex operations as you need them.

4. **Test.** Write test cases for each unit (ADT or method).

5. **Implement simply first** Choose simple,  brute-force representations. The point here is to put pressure on the specs and the tests, and try to pull your whole program together as soon as possible. Make the whole program work correctly first. Skip the advanced features for now. Skip performance optimization. Skip corner cases. Keep a to-do list of what you have to revisit.

6. **Iterate.** Now that it's all working, make it work better. Reimplement, optimize, redesign if necessary.
