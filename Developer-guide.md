Here are some ways people might want to extend Hammer. 

## Adding new combinators
* Combinators are declared in src/hammer.h and defined in src/parsers/
* Declaration is done with the `HAMMER_FN_DECL` family of macros (see hammer.h)
* What gets declared is a function that returns an `HParser*`; this function needs to call a corresponding `__m` function that calls `h_new_parser` to instantiate an `HParser*` with the combinator's vtable. Example, suitable for boilerplate:

        HParser* h_foo(const int n) {
            return h_foo__m(&system_allocator, n);
        }
        HParser* h_foo__m(HAllocator* mm__, const int n) {
            return h_new_parser(mm__, &foo_vt, (void*)(intptr_t)n);
        }
h_new_parser expects its third argument to be a void*; if a combinator has more than one argument, the combinator implementation should also define a struct to hold the arguments (referred to as the "environment"). See src/parsers/token.c for a simple example.
* Each vtable is a struct that must define the following members, all of which are function pointers:
  * `.parse`
    * Signature: `static HParseResult* parse_foo(void *env, HParseState *state)`
    * destructures the environment that was passed in (if necessary)
    * if a primitive combinator (e.g. `h_ch`), consumes input directly off the input stream using `h_read_bits`
    * if a higher-order combinator (e.g. `h_sequence`), applies its component parsers to the input stream using `h_do_parse`
    * constructs result `HParsedToken*` and wraps it in an `HParseResult*`
  * `.isValidRegular`
    * for primitive combinators, this will just be `h_true`
    * for higher-order combinators that can never be regular (e.g. `h_indirect`), use `h_false`
    * for higher-order combinators that need a function to determine whether their expansion is regular (e.g. `h_sequence`), the signature is `static bool foo_isValidRegular(void *env)`
  * `.isValidCF`
    * for primitive combinators, this will just be `h_true`
    * for higher-order combinators that can never be context-free (e.g. `h_length_value`), use `h_false`
    * for higher-order combinators that need a function to determine whether their expansion is context-free (e.g. `h_choice`), the signature is `static bool foo_isValidCF(void *env)`
  * `.desugar`
    * Signature: `static void desugar_foo(HAllocator *mm__, HCFStack *stk, void *env)`
    * converts an `HParser*` to a sum-of-products representation for the context-free backends
    * there are some macros to make this easier, defined in src/backends/contextfree.h
    * doesn't need to be defined if `.isValidCF = h_false`
  * `.compile_to_rvm`
    * Signature: `static void foo_ctrvm(HRVMProg *prog, void *env)`
    * converts an `HParser*` to a corresponding set of instructions for the regex VM. Those are defined in src/backends/regex.h.

## Adding new parsing backends
* declare a new `extern HParserBackendVTable` in internal.h with the existing ones
* add it to the `HParserBackend` enum in hammer.h
* add it to `*backends` in hammer.c
* the vtable in your backend implementation is a struct which must define the following members, all function pointers:
  * `.compile`
    * Signature: `int h_bar_compile(HAllocator* mm__, HParser* parser, const void* params)`
    * does any setup work for using this backend (e.g., generating parse tables)
    * returns 0 on success, -1 on failure
  * `.parse`
    * Signature: `HParseResult *h_bar_parse(HAllocator* mm__, const HParser* parser, HInputStream* stream)`
  * `.free`
    * Signature: `void h_bar_free(HParser *parser)`
    * for cleaning up anything that `.compile` created

## Adding new language bindings