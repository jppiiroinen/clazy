# rule-of-two-soft

Finds places where:
1. You're calling a trivial copy-ctor of a class which has a non-trivial copy-assignment operator
2. You're calling a trivial copy-assignment operator of a class which has a non-trivial copy-ctor

It won't warn on classes that violate the rule of two unless you actually use the copy-ctor or the copy-assignment operator, otherwise it will generate lots of warnings. If you're really interested in all warnings, see the *rule-of-three* check instead of *rule-of-two-soft*.

This check is safe with no false-positives, but is disabled by default: beware that removing copy-ctors or copy-assignment operators might make the class trivially copiable, which is not ABI compatible.
