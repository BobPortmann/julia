.. highlight:: c

***********************************
Initialization of the Julia runtime
***********************************

How does the Julia runtime execute :code:`julia -e 'println("Hello World!")'` ?

main()
------

Execution starts at `main() in julia/ui/repl.c
<https://github.com/JuliaLang/julia/blob/master/ui/repl.c#L333>`_.

main() calls `libsupport_init()
<https://github.com/JuliaLang/julia/blob/master/src/support/libsupportinit.c#L10>`_
to set the C library locale and to initialise the "ios" library
(see `ios_init_stdstreams()
<https://github.com/JuliaLang/julia/blob/master/src/support/ios.c#L917>`_
and :ref:`dev-ios`).

Next `parse_opts()
<https://github.com/JuliaLang/julia/blob/master/ui/repl.c#L80>`_
is called to process command line options. Note that :c:func:`parse_opts`
only deals with options that affect code generation or early initialisation. Other
options are handled later by `process_options() in base/client.jl
<https://github.com/JuliaLang/julia/blob/master/base/client.jl#L214>`_.

:c:func:`parse_opts` stores command line options in the `global jl_compileropts
struct
<https://github.com/JuliaLang/julia/blob/master/src/julia.h#L1320>`_.


julia_init()
------------


`julia_init() in task.c
<https://github.com/JuliaLang/julia/blob/master/src/task.c#L270>`_ is
called by main() and calls `_julia_init() in init.c
<https://github.com/JuliaLang/julia/blob/master/src/init.c#L875>`_.

:c:func:`_julia_init` begins by calling :c:func:`libsupport_init` again (it does
nothing the second time).

`restore_signals()
<https://github.com/JuliaLang/julia/blob/master/src/init.c#L402>`_ is
called to zero the signal handler mask.

`jl_resolve_sysimg_location()
<https://github.com/JuliaLang/julia/blob/master/src/init.c#L823>`_ searches
configured paths for the base system image. See :ref:`dev-sysimg`.

`jl_gc_init()
<https://github.com/JuliaLang/julia/blob/master/src/gc.c#L1096>`_
sets up allocation pools and lists for: weak refs, preserved values
and finalization.

`jl_init_frontend()
<https://github.com/JuliaLang/julia/blob/master/src/ast.c#L119>`_
loads and initialises a pre-compiled femtolisp image containing
the scanner/parser;

`jl_init_types()
<https://github.com/JuliaLang/julia/blob/master/src/jltypes.c#L2887>`_
creates :c:type:`jl_datatype_t` type description objects for the `built-in
types defined in julia.h
<https://github.com/JuliaLang/julia/blob/master/src/julia.h#L295>`_. e.g.
::

    jl_any_type = jl_new_abstracttype(jl_symbol("Any"), NULL, jl_null);
    jl_any_type->super = jl_any_type;

    jl_type_type = jl_new_abstracttype(jl_symbol("Type"), jl_any_type, jl_null);

    jl_int32_type = jl_new_bitstype(jl_symbol("Int32"),
                                    jl_any_type, jl_null, 32);

`jl_init_tasks()
<https://github.com/JuliaLang/julia/blob/master/src/task.c#L870>`_ creates
the ``jl_datatype_t* jl_task_type`` object; initialises the global
`jl_root_task struct
<https://github.com/JuliaLang/julia/blob/master/src/julia.h#L1159>`_; and
sets ``jl_current_task`` to the root task.

`jl_init_codegen()
<https://github.com/JuliaLang/julia/blob/master/src/codegen.cpp#L4830>`_
initialises the `LLVM library <http://llvm.org>`_.

`jl_init_serializer()
<https://github.com/JuliaLang/julia/blob/master/src/dump.c#L1732>`_
initialises 8-bit serialisation tags for 256 frequently used
``jl_value_t`` values. The serialisation mechanism uses these tags as
shorthand (in lieu of storing whole objects) to save storage space.

.. sidebar:: sysimg

    If there is a sysimg file, it contains a pre-cooked image of the :mod:`Core` and :mod:`Main` modules (and whatever else is created by ``boot.jl``). See :ref:`dev-sysimg`.

    `jl_restore_system_image() <https://github.com/JuliaLang/julia/blob/master/src/dump.c#L1379>`_ de-serialises the saved sysimg into the current Julia runtime environment and initialisation continues after :c:func:`jl_init_box_caches` below...

    Note: `jl_restore_system_image() (and dump.c in general) <https://github.com/JuliaLang/julia/blob/master/src/dump.c#L1379>`_ uses the :ref:`dev-ios`.


If there is no sysimg file (:code:`!jl_compileropts.image_file`) then
then :mod:`Core` and :mod:`Main` modules are created and ``boot.jl`` is evaluated:

:code:`jl_core_module = jl_new_module(jl_symbol("Core"))` creates
the Julia :mod:`Core` module.

`jl_init_intrinsic_functions()
<https://github.com/JuliaLang/julia/blob/master/src/intrinsics.cpp#L1254>`_
creates a new Julia module "Intrinsics" containing constant
jl_intrinsic_type symbols. These define an integer code for
each `intrinsic function
<https://github.com/JuliaLang/julia/blob/master/src/intrinsics.cpp#L2>`_.
`emit_intrinsic()
<https://github.com/JuliaLang/julia/blob/master/src/intrinsics.cpp#L757>`_
translates these symbols into LLVM instructions during code generation.

`jl_init_primitives()
<https://github.com/JuliaLang/julia/blob/master/src/builtins.c#L989>`_
hooks C functions up to Julia function symbols. e.g. the symbol
:func:`Base.is` is bound to C function pointer :c:func:`jl_f_is`
by calling :code:`add_builtin_func("eval", jl_f_top_eval)`, which does::

    jl_set_const(jl_core_module,
                 jl_symbol("is"),
                 jl_new_closure(jl_f_top_eval, jl_symbol("eval"), NULL));


`jl_new_main_module()
<https://github.com/JuliaLang/julia/blob/master/src/toplevel.c>`_
creates the global "Main" module and sets
:code:`jl_current_task->current_module = jl_main_module`.

Note: _julia_init() `then sets <https://github.com/JuliaLang/julia/blob/master/src/init.c#L975>`_ :code:`jl_root_task->current_module = jl_core_module`. :code:`jl_root_task` is an alias of :code:`jl_current_task` at this point, so the current_module set by :c:func:`jl_new_main_module` above is overwritten.

`jl_load("boot.jl") <https://github.com/JuliaLang/julia/blob/master/src/toplevel.c#L568>`_ calls `jl_parse_eval_all("boot.jl") <https://github.com/JuliaLang/julia/blob/master/src/toplevel.c#L525>`_ which repeatedly calls `jl_parse_next() <https://github.com/JuliaLang/julia/blob/master/src/ast.c#L523>`_ and `jl_toplevel_eval_flex() <https://github.com/JuliaLang/julia/blob/master/src/toplevel.c#L376>`_ to parse and execute `boot.jl <https://github.com/JuliaLang/julia/blob/master/base/boot.jl#L116>`_. TODO -- drill down into eval?

`jl_get_builtin_hooks() <https://github.com/JuliaLang/julia/blob/master/src/init.c#L1209>`_ initialises global C pointers to Julia globals defined in ``boot.jl``.


`jl_init_box_caches() <https://github.com/JuliaLang/julia/blob/master/src/alloc.c#L850>`_ pre-allocates global boxed integer value objects for values up to 1024. This speeds up allocation of boxed ints later on. e.g.::

    jl_value_t *jl_box_uint8(uint32_t x)
    {
        return boxed_uint8_cache[(uint8_t)x];
    }

`_julia_init() iterates <https://github.com/JuliaLang/julia/blob/master/src/init.c#L997>`_ over the :code:`jl_core_module->bindings.table` looking for :code:`jl_datatype_t` values and sets the type name's module prefix to :code:`jl_core_module`.

`jl_add_standard_imports(jl_main_module) <https://github.com/JuliaLang/julia/blob/master/src/toplevel.c#L34>`_ does "using Base" in the "Main" module.

Note: :c:func:`_julia_init` `now reverts <https://github.com/JuliaLang/julia/blob/master/src/init.c#L1017>`_ to :code:`jl_root_task->current_module = jl_main_module` as it was before being `set to jl_core_module <https://github.com/JuliaLang/julia/blob/master/src/init.c#L975>`_ above.

Platform specific signal handlers are initialised for ``SIGSEGV`` (OSX, Linux), and ``SIGFPE`` (Windows).

Other signals (``SIGINFO, SIGBUS, SIGILL, SIGTERM, SIGABRT, SIGQUIT, SIGSYS`` and ``SIGPIPE``) are hooked up to `sigdie_handler() <https://github.com/JuliaLang/julia/blob/master/src/init.c#L174>`_ which prints a backtrace.

`jl_init_restored_modules() <https://github.com/JuliaLang/julia/blob/master/src/dump.c#L1458>`_ calls `jl_module_run_initializer() <https://github.com/JuliaLang/julia/blob/master/src/module.c#L429>`_ for each deserialised module to run the :c:func:`__init__` function.

Finally `sigint_handler() <https://github.com/JuliaLang/julia/blob/master/src/init.c#L409>`_ is hooked up to ``SIGINT`` and calls :code:`jl_throw(jl_interrupt_exception)`.

:c:func:`_julia_init` then returns `back to main() in julia/ui/repl.c
<https://github.com/JuliaLang/julia/blob/master/ui/repl.c#L355>`_ and main() calls :code:`true_main(argc, (char**)argv)`.

true_main()
-----------

`true_main() <https://github.com/JuliaLang/julia/blob/master/ui/repl.c#L275>`_ loads the contents of :code:`argv[]` into :data:`Base.ARGS`.

If a .jl "program" file was supplied on the command line, then `exec_program() <https://github.com/JuliaLang/julia/blob/master/ui/repl.c#L219>`_ calls `jl_load(program) <https://github.com/JuliaLang/julia/blob/master/src/toplevel.c#L568>`_ which calls `jl_parse_eval_all() <https://github.com/JuliaLang/julia/blob/master/src/toplevel.c#L525>`_ which repeatedly calls `jl_parse_next() <https://github.com/JuliaLang/julia/blob/master/src/ast.c#L523>`_ and `jl_toplevel_eval_flex() <https://github.com/JuliaLang/julia/blob/master/src/toplevel.c#L376>`_ to parse and execute the program.

However, in our example (:code:`julia -e 'println("Hello World!")'`), `jl_get_global(jl_base_module, jl_symbol("_start")) <https://github.com/JuliaLang/julia/blob/master/src/module.c#L320>`_ looks up `Base._start <https://github.com/JuliaLang/julia/blob/master/base/client.jl#L388>`_ and `jl_apply() <https://github.com/JuliaLang/julia/blob/master/src/julia.h#L987>`_ executes it.


Base._start
-----------

`Base._start <https://github.com/JuliaLang/julia/blob/master/base/client.jl#L388>`_ calls `Base.process_options <https://github.com/JuliaLang/julia/blob/master/base/client.jl#L214>`_ which calls `jl_parse_input_line("println(\"Hello World!\")") <https://github.com/JuliaLang/julia/blob/master/src/ast.c#L468>`_ to create an expression object and :func:`Base.eval` to execute it.


Base.eval
---------

:func:`Base.eval` was `mapped to jl_f_top_eval <https://github.com/JuliaLang/julia/blob/master/src/builtins.c#L1005>`_ by :c:func:`jl_init_primitives`.

`jl_f_top_eval() <https://github.com/JuliaLang/julia/blob/master/src/builtins.c#L444>`_ calls `jl_toplevel_eval_in(jl_main_module, ex) <https://github.com/JuliaLang/julia/blob/master/src/builtins.c#L444>`_, where "ex" is the parsed expression :code:`println("Hello World!")`.

`jl_toplevel_eval_in() <https://github.com/JuliaLang/julia/blob/master/src/builtins.c#L417>`_ calls `jl_toplevel_eval_flex() <https://github.com/JuliaLang/julia/blob/master/src/toplevel.c#L376>`_ which calls `eval() in interpreter.c <https://github.com/JuliaLang/julia/blob/master/src/interpreter.c#L112>`_.

The stack dump below shows how the interpreter works its way through various methods of :func:`Base.println` and :func:`Base.print` before arriving at `write{T}(s::AsyncStream, a::Array{T}) <https://github.com/JuliaLang/julia/blob/master/base/stream.jl#L782>`_  which does :code:`ccall(jl_write_no_copy())`.

`jl_write_no_copy() <https://github.com/JuliaLang/julia/blob/master/src/jl_uv.c#L580>`_
calls :c:func:`uv_write` to write "Hello World!" to :c:macro:`JL_STDOUT`. See :ref:`dev-libuv`.::

    Hello World!


============================  =================  ===============================================
Stack frame                   Source code        Notes
============================  =================  ===============================================
jl_write_no_copy()            jl_uv.c:552        called though :func:`Base.ccall`
julia_write_282942            stream.jl:734      function write!{T}(s::AsyncStream, a::Array{T})
julia_print_284639            ascii.jl:93        print(io::IO, s::ASCIIString) = (write(io, s);nothing)
jlcall_print_284639
jl_apply()                    julia.h:989
jl_trampoline()               builtins.c:835
jl_apply()                    julia.h:989
jl_apply_generic()            gf.c:1624          Base.print(Base.TTY, ASCIIString)
jl_apply()                    julia.h:989
jl_trampoline()               builtins.c:835
jl_apply()                    julia.h:989
jl_apply_generic()            gf.c:1643          Base.print(Base.TTY, ASCIIString, Char, Char...)
jl_apply()                    julia.h:989
jl_f_apply()                  builtins.c:374
jl_apply()                    julia.h:989
jl_trampoline()               builtins.c:835
jl_apply()                    julia.h:989
jl_apply_generic()            gf.c:1643          Base.println(Base.TTY, ASCIIString, ASCIIString...)
jl_apply()                    julia.h:989
jl_trampoline()               builtins.c:835
jl_apply()                    julia.h:989
jl_apply_generic()            gf.c:1643          Base.println(ASCIIString,)
jl_apply()                    julia.h:989
do_call()                     interpreter.c:70
eval()                        interpreter.c:210
jl_interpret_toplevel_expr()  interpreter.c:25
jl_toplevel_eval_flex()       toplevel.c:498
jl_toplevel_eval()            toplevel.c:521
jl_toplevel_eval_in()         builtins.c:440
jl_f_top_eval()               builtins.c:469
============================  =================  ===============================================

Since our example has just one function call, which has done its
job of printing "Hello World!", the stack now rapidly unwinds back to :c:func:`main`.

jl_atexit_hook()
----------------

:c:func:`main` calls `jl_atexit_hook()
<https://github.com/JuliaLang/julia/blob/master/src/init.c#L448>`_. This
calls _atexit for each module, then calls `jl_gc_run_all_finalizers()
<https://github.com/JuliaLang/julia/blob/master/src/gc.c#L325>`_
and cleans up libuv handles.


julia_save()
------------

Finally, :c:func:`main` calls `julia_save() <https://github.com/JuliaLang/julia/blob/master/src/init.c#L1155>`_, which if requested on the command line, saves the runtime state to a new system image. See `jl_compile_all() <https://github.com/JuliaLang/julia/blob/master/src/gf.c#L1525>`_ and `jl_save_system_image() <https://github.com/JuliaLang/julia/blob/master/src/dump.c#L1300>`_.
