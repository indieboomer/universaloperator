# Exact Universal Operator Search Lab

Build a Python research tool for searching candidate exact universal binary operators.

Goal:
Find binary operators o(x, y), possibly with one allowed constant c, that can exactly generate basic arithmetic operations through finite expression trees built only from:
- variables x, y
- one constant, usually 0 or 1
- the operator o(x, y)

We are NOT looking for numerical approximation in this version.
This version searches for exact symbolic definability.

Core research question:
Can a single binary operator o(x, y), plus one constant, generate:
- negation: -x
- identity: x
- addition: x + y
- subtraction: x - y
- multiplication: x * y
- constants: 0, 1, -1
- possibly division or reciprocal later

This is inspired by operators like:

    eml(x, y) = exp(x) - log(y)

and especially:

    hml(x, y) = sinh(x) - asinh(y)

For hml with constant 0, we found exact formulas for negation, subtraction, and addition, but multiplication is blocked by odd symmetry. This tool should generalize that kind of search.

------------------------------------------------------------
1. Project structure
------------------------------------------------------------

Create a Python package:

    exact_operator_lab/
        __init__.py
        operators.py
        expression.py
        generator.py
        targets.py
        symbolic_check.py
        numeric_filter.py
        search.py
        symmetry.py
        report.py
        experiments/
            hml_exact.py
            eml_exact.py
            candidate_search.py
        tests/
            test_expression.py
            test_hml.py
            test_symmetry.py
            test_search.py
        README.md

Use:
- Python 3.11+
- sympy
- numpy
- pandas
- rich
- pytest

No heavy ML libraries.

------------------------------------------------------------
2. Core definitions
------------------------------------------------------------

Represent a candidate operator as:

    class BinaryOperator:
        name: str
        expr: sympy.Expr
        x: sympy.Symbol
        y: sympy.Symbol
        domain_notes: str | None

Example:

    o(x,y) = sinh(x) - asinh(y)

    o = BinaryOperator(
        name="hml",
        expr=sinh(x) - asinh(y)
    )

The system must also allow operators defined from strings:

    "sinh(x) - asinh(y)"
    "exp(x) - log(y)"
    "x/y"
    "x - y + 1"
    "x**2 - sqrt(y)"
    "tan(x) - atan(y)"

------------------------------------------------------------
3. Expression trees
------------------------------------------------------------

Expression grammar:

    E ::= x | y | c | o(E, E)

where c is a single allowed constant, e.g. 0 or 1.

Represent expression trees separately from SymPy expressions so we can:
- track tree size
- track depth
- print formulas in operator notation
- convert to SymPy for checking

Example expression tree:

    o(o(c, x), o(y, c))

Should print as:

    o(o(0, x), o(y, 0))

and convert to SymPy by recursively substituting:

    o(a,b) = operator.expr.subs({x:a, y:b})

------------------------------------------------------------
4. Generation strategy
------------------------------------------------------------

Generate all unique expressions up to max depth.

Modes:

A. Exhaustive small search
    max_depth <= 4 or 5
    useful for exact formulas

B. Size-limited search
    max_nodes <= N

C. Beam search
    keep only expressions that are numerically close to targets

Deduplication:
- Convert to SymPy
- simplify
- use srepr or string form as key
- also use numeric fingerprints to catch equivalent expressions that SymPy does not simplify

Important:
Avoid combinatorial explosion.

At each depth:
- generate o(a,b) from previous expressions
- simplify
- discard if too large
- discard if contains NaN-prone undefined expression for sampled points
- keep only unique expressions

------------------------------------------------------------
5. Target operations
------------------------------------------------------------

The tool should search for exact definitions of:

Unary targets:
    ID_X: x
    NEG_X: -x
    ZERO: 0
    ONE: 1
    MINUS_ONE: -1

Binary targets:
    ADD: x + y
    SUB: x - y
    RSUB: y - x
    MUL: x * y

Later:
    SQUARE_X: x**2
    RECIP_X: 1/x
    DIV: x/y

Each target should have:
- name
- SymPy expression
- arity: unary or binary
- test variables

For unary targets, expressions may use x and constant c.
For binary targets, expressions may use x, y and constant c.

------------------------------------------------------------
6. Exact symbolic checking
------------------------------------------------------------

For candidate expression E and target T:

1. Compute:
       diff = simplify(E - T)

2. If diff == 0:
       exact match

3. If not, try stronger simplification:
       trigsimp
       powsimp
       expand
       factor
       cancel
       together
       simplify again

4. For hyperbolic functions:
       use expand_func
       rewrite(exp)
       simplify

5. Numeric sanity check:
       evaluate diff on random rational points
       if always near zero, mark as "suspected exact but not proven"

Report status:
- EXACT
- NUMERIC_MATCH_SYMBOLIC_FAILED
- NO_MATCH

------------------------------------------------------------
7. Numeric prefilter
------------------------------------------------------------

Before expensive symbolic comparison, evaluate expressions on sample points.

For binary:
    sample x,y from:
        [-2,-1,-0.5,0,0.5,1,2]
        plus random rational points

Discard expression-target candidate if:
    max_abs_error > tolerance

Use tolerance:
    1e-8 for exact candidates

This is only a filter, not final proof.

------------------------------------------------------------
8. Symmetry analysis
------------------------------------------------------------

Very important.

Implement automatic symmetry obstruction checks.

For operator o and constant c:

Check whether all generated expressions must satisfy a parity property.

Example:
For hml:

    o(x,y) = sinh(x) - asinh(y)

with constant 0:

    o(-x,-y) = -o(x,y)

Therefore every expression generated from x,y,0 is odd under global sign flip:

    E(-x,-y) = -E(x,y)

This blocks multiplication:

    (−x)(−y) = xy

because multiplication is even under global sign flip.

Implement:

    detect_global_odd_symmetry(operator, constant)

Check:
    o(-x,-y) + o(x,y) simplifies to 0
    constant c satisfies -c = c, i.e. c = 0

If true:
    generated expressions are globally odd
    targets like x*y, x^2, y^2, 1 are impossible

Report obstruction.

Also test:
- symmetry: o(x,y)=o(y,x)
- antisymmetry: o(x,y)=-o(y,x)
- constant generation: o(c,c)
- self-constant: o(x,x)

------------------------------------------------------------
9. Candidate operator families
------------------------------------------------------------

Implement a candidate generator for operator forms.

Start with manually defined operators:

A. Inverse-pair operators:

    o_A(x,y) = A(x) - A_inverse(y)

Examples:
    sinh(x) - asinh(y)
    exp(x) - log(y)
    x**3 - real_cuberoot(y)
    tan(x) - atan(y)
    x*exp(x) - LambertW(y)

B. Shifted inverse-pair operators:

    o_A,c(x,y) = A(x) - A_inverse(y) + c

Examples:
    sinh(x) - asinh(y) + 1
    exp(x) - log(y) + 1

C. Self-constant operators:

    x/y
    exp(x-y)
    x-y+1
    (x+1)/(y+1)

D. Polynomial/rational operators:

    x+y
    x-y
    x*y
    x/y
    x+y+x*y
    x-y+x*y
    x**2-y
    x-y**2

The tool should allow a grammar-based search later, but first implement manual candidates.

------------------------------------------------------------
10. Search procedure
------------------------------------------------------------

For each operator and constant c:

1. Analyze operator:
    - domain notes
    - o(c,c)
    - o(x,x)
    - symmetries
    - possible obstruction to targets

2. Generate expressions up to depth D.

3. For each target:
    - numeric prefilter
    - symbolic exact check
    - record first/best expression

4. Report:
    - found exact formulas
    - suspected formulas
    - impossible by symmetry
    - not found within depth

5. Score operator.

------------------------------------------------------------
11. Scoring exact universality
------------------------------------------------------------

Score candidate operator:

    +10 exact ZERO
    +10 exact ONE
    +10 exact NEG_X
    +15 exact ADD
    +15 exact SUB
    +25 exact MUL
    +10 exact SQUARE_X
    - penalty for requiring deeper trees
    - penalty for domain restrictions
    - penalty for expression explosion

Operator is "exact universal arithmetic candidate" if it generates:
- 0
- 1
- negation
- addition
- multiplication

Subtraction follows from addition + negation, but direct formula is also useful.

------------------------------------------------------------
12. Reports
------------------------------------------------------------

Generate Markdown report per operator.

Example:

    # Operator hml

    Definition:
        o(x,y) = sinh(x) - asinh(y)

    Constant:
        c = 0

    Symmetry:
        o(-x,-y) = -o(x,y)
        constant 0 preserves odd symmetry
        Therefore all generated expressions are globally odd.
        Multiplication x*y is impossible in this system.

    Found exact expressions:

        neg(x) = o(0, o(x,0)) = -x

        sub(x,y) = ...
        add(x,y) = ...

    Not found / impossible:
        x*y impossible by parity obstruction.

    Score:
        ...

------------------------------------------------------------
13. Required baseline: hml with c=0
------------------------------------------------------------

The program must reproduce known formulas for:

    o(x,y)=sinh(x)-asinh(y)

with c=0.

Expected formulas:

    S(x) = o(x,0) = sinh(x)

    neg(x) = o(0, o(x,0)) = -x

because:
    o(0, sinh(x)) = -asinh(sinh(x)) = -x

    asinh(x) can be built as:
        P(x) = o(0, o(o(0,x),0))

Check this carefully.

Then:

    sub(a,b) = a-b

Known compact formula from prior reasoning:

    a-b =
    o(
        o(0, o(o(0,a),0)),
        o(b,0)
    )

The program should verify it symbolically.

Addition:

    a+b =
    o(
        o(0, o(o(0,a),0)),
        o(o(0,o(b,0)),0)
    )

The program should verify it symbolically.

Also prove/report:
    multiplication x*y impossible due to global odd symmetry.

------------------------------------------------------------
14. Baseline: hml with c=0 and square extension later
------------------------------------------------------------

Do not implement this in the main exact binary-only search, but include a note:
If unary square S(x)=x^2 is added, multiplication follows by polarization:

    xy = ((x+y)^2 - x^2 - y^2)/2

This belongs to a later "extended primitives" version.

------------------------------------------------------------
15. CLI
------------------------------------------------------------

Implement commands:

    python -m exact_operator_lab analyze --operator "sinh(x)-asinh(y)" --constant 0 --name hml

    python -m exact_operator_lab search --operator "sinh(x)-asinh(y)" --constant 0 --depth 5

    python -m exact_operator_lab batch --preset inverse_pairs --constant 0 --depth 4

    python -m exact_operator_lab report --input results/hml.json --output reports/hml.md

------------------------------------------------------------
16. Tests
------------------------------------------------------------

Write pytest tests:

1. Expression tree conversion works.
2. hml negation formula simplifies to -x.
3. hml subtraction formula simplifies to x-y.
4. hml addition formula simplifies to x+y.
5. global odd symmetry detection works for hml + 0.
6. multiplication is marked impossible for hml + 0.
7. associativity check works for o(x,y)=x+y.
8. self-constant check works for o(x,y)=x/y with c not needed:
       o(x,x)=1

------------------------------------------------------------
17. Deliverable
------------------------------------------------------------

Build a working prototype, not just skeleton code.

Minimum successful deliverable:
- define operators
- generate expression trees up to depth 4
- detect exact symbolic matches to target operations
- reproduce hml addition/subtraction/negation formulas
- detect parity obstruction for multiplication
- generate a Markdown report

Prioritize clarity and correctness over speed.
