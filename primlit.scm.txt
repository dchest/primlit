; ================================================
; Primitive literate programming system for Scheme
; ================================================
;
; :Author: Dmitry Chestnykh
; :Version: 0.4 (2011-11-29)
; :License: `ISC <https://en.wikipedia.org/wiki/ISC_license>`_
;
; .. contents::
; 
; Introduction
; ------------
;
; This almost `literate programming`_ system uses reStructuredText_ inside line
; comments in Scheme code for writing, and normal Scheme code as a program. That
; is, the source code is runnable without conversion, but generating the book
; requires conversion. It is similar to Docco_.
;
; .. _literate programming: https://en.wikipedia.org/wiki/Literate_programming 
; .. _reStructuredText:     http://docutils.sourceforge.net/rst.html
; .. _Docco:                http://jashkenas.github.com/docco/
;
; This program prepares source code for feeding it into reStructuredText tools,
; such as `rst2html`. Basically, it extracts text from comments by stripping ';'
; from the beginning, and transforms program code into code blocks with a few
; added CSS classes (`program` and `language-scheme`).
;
; The rules are simple:
;
; * text blocks are separated from program code with blank lines.
; * lines of text blocks begin with semicolon and space: ``"; "``.
; * everything inside text blocks is formatted with reStructuredText.
; * everything between text blocks is program code.
; 
; Example
; -------
; ::
;
;   ; This is a comment and a *text block*.
;   ; It spans five lines, after it goes a blank line, and program code.
;   ;
;   ; The following procedure :proc:`greeting` accepts variable :var:`x`,
;   ; and displays the greeting.
;
;   (define (greeting x)
;     (display (string-append "Greetings from program code, " x)))
;
;   ; Text continues here...
;
; For another example, view the `source code for this document
; <http://www.codingrobots.org/primlit/primlit.scm.txt>`_.
;
; Usage
; -----
;
; This program reads from standard input and writes to standard output.
; Run it on your source and then feed the output to `rst2html`, for example::
;
;   $ guile -s primlit.scm < your_source.scm | rst2html > output.html
;
;
; The Program
; -----------
;
; Text Processing
; ~~~~~~~~~~~~~~~
;
; Text lines (lines that are not program code) begin with semicolon.  Let's
; write a function to distinguish such lines. It will accept a string :var:`s`
; and return :value:`#t` if it's a text line.

(define (text-line? s)
  (and (> (string-length s) 0)
       (char=? (string-ref s 0) #\;)))

; To output text lines, we should strip semicolon and maybe a space from it.
; The following function does exactly this, and outputs the result:

(define (output-text-line s)
  (display
    (substring s
               (if (and (>= (string-length s) 2)
                        (char=? (string-ref s 1) #\ ))
                 2  ; cut two characters, semicolon and space
                 1) ; cut one character, semicolon
               (string-length s))))

; Outputting code lines is easier, we just have to keep in mind that code blocks
; must be indented by two spaces for reStructuredText to format them correctly.

(define (output-code-line s)
  (display (string-append "  " s)))
  
; Styling
; ~~~~~~~
;
; To distinguish program code from other ``pre`` blocks in output, we'll assign
; special classes to ``pre`` blocks with code.
;
; This functions outputs a special markup to let reStructuredText know that it
; needs to add ``program`` and ``language-scheme`` to the classes of ``pre``
; element that wraps the program code.

(define (start-code-block)
  (display "\n.. class :: program language-scheme\n\n::\n\n  "))

; Default styles shipped with reStructuredText tools don't know about such
; classes, so we need to add our own style. Instead of creating templates, let's
; just put this quick hack, which only works with ``rst2html``, but not
; ``rst2latex`` and other tools -- a CSS style block that makes program code
; appear in navy color and adds a thick vertical border to the left.
;
; We'll write this directly, so that it appears before any other output:

(display ".. raw:: html

   <style>
   pre.program { 
       color: navy;
       border-left: 8px solid #eee;
       padding-left: 16px;
       margin-left: 0;
   }
   </style>

")

; Additional Markup
; ~~~~~~~~~~~~~~~~~
;
; When we write a variable, a procedure or a macro name in the text, we'd like
; them to appear as literal text. It would also be helpful to assign classes to
; their markup. Let's output directives for reStructuredText:

(display
"
.. role:: proc(literal)
   :class: procedure

.. role:: var(literal)
   :class: variable

.. role:: value(literal)
   :class: value

.. role:: macro(literal)
   :class: macro

.. role:: module(literal)
   :class: module

")

; Main Loop
; ~~~~~~~~~
;
; Finally, the read-write loop. We'll read lines from the standard input until
; we reach the end of file, and process them one-by-one.
;
; We'll use :proc:`read-line` function from :module:`rdelim` module and R6RS
; control structures, so let's import these modules:

(use-modules (ice-9 rdelim)
             (rnrs control))
;
; In the loop we read a line from input into variable :var:`s`, and also
; remember the last line read in the variable :var:`prev`. We need :var:`prev`
; in order to know where to start a code block (if the previous line was a text
; line, and the current one is not, then we should start a code block).

(let loop ((s (read-line)) (prev ""))
  (unless (eof-object? s)
    (cond
      ((text-line? s) (output-text-line s))
      (else
        (when (text-line? prev) (start-code-block))
        (output-code-line s)))
    (newline)
    (loop (read-line) s)))

; That's it! 
;
; `Download source code <http://www.codingrobots.org/primlit/primlit.scm>`_
; and run it with `Guile <http://www.gnu.org/software/guile/>`_.
