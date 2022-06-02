# NOTE: The MEDLEY-INIT greet file in the Medley repository loads BACKGROUND-YIELD now, so no user action is needed.

Interlisp-D was originally designed with a "busy wait" scheduler, which keeps the Lisp process busy at all times.
On some machine configurations, this isn't so good.

Support is in the latest Maiko.

Soon this will be in the standard Medley release, but until then, load BACKGROUND-YIELD.LCOM in your init.
```
(ADDTOFILE 'BACKGROUND-YIELD 'FILES 'INIT)
``` 
