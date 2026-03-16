---
name: questions-and-assessments
description: >
  Create Wolfram Language QuestionObject and AssessmentFunction expressions. Use this skill
  whenever the user wants to build a quiz question, assessment, multiple-choice item, true/false
  question, short-answer grader, numeric question, fill-in-the-blank, or any kind of interactive
  question for a Wolfram notebook or course. Also trigger when the user asks about scoring,
  partial credit, explanation feedback, comparison methods (AlgebraicValue, CalculusResult,
  CodeEquivalence, etc.), QuestionGenerator, or QuestionInterface types.
  Trigger even if they just say "make a question about X" or "how do I grade answers in WL".
---

# Questions and Assessment in Wolfram Language

This skill guides you through constructing correct, idiomatic `QuestionObject` and
`AssessmentFunction` expressions — the core building blocks of the Wolfram Language assessment
framework (System context, experimental).

QuestionObject should only be used when you want to create an interactive question interface for use in a Wolfram Notebook.

AssessmentFunction is purely computational. It is used to determine if answers match an answer key.

---

## Core Structure

```wl
QuestionObject[prompt, AssessmentFunction[key]]
QuestionObject[prompt, AssessmentFunction[key, method]]
QuestionObject[prompt, AssessmentFunction[key, <|settings...|>]]
```

`QuestionObject[assess]` (no explicit prompt) derives a prompt automatically from the key.

---

## AssessmentFunction Key Syntax

The `key` specifies correct and incorrect answers. Each entry can be:

| Form | Meaning |
|---|---|
| `ans` | correct answer (score 1) |
| `ans -> score` | answer with explicit score (positive = correct) |
| `ans -> True` / `ans -> False` | correct / incorrect shorthand |
| `ans -> <\|"Score"->s, "Explanation"->"...", "AnswerCorrect"->True\|>` | full answer spec |

**Single correct answer from a list** — give one entry a positive score; everything else becomes incorrect automatically:
```wl
AssessmentFunction[{1, 2, 7 -> 1, 9, 10}]
(* Only 7 is correct *)
```

**All answers correct** — omit scores entirely:
```wl
AssessmentFunction[{Pi, 3.14159}]  (* both accepted *)
```

**Explicit True/False scores:**
```wl
AssessmentFunction[{755 -> True, 714 -> False, 868 -> False}]
```

**Full answer specification with per-answer feedback:**
```wl
AssessmentFunction[{
  "Paris" -> <|"Score" -> 1, "Explanation" -> "Correct! Paris is the capital of France."|>,
  "Lyon"  -> <|"Score" -> 0, "Explanation" -> "Lyon is France's second largest city, not the capital."|>,
  "Nice"  -> <|"Score" -> 0, "Explanation" -> "Nice is a city in the French Riviera."|>
}]
```

---

## Interface Types

The interface is usually inferred automatically from the key, but can be set explicitly with
`QuestionInterface[type, <|properties|>]`.

| Interface | Auto-inferred when… | Notes |
|---|---|---|
| `"MultipleChoice"` | list key, exactly one correct | Radio buttons |
| `"ChooseMultiple"` | list key, multiple correct + `"SeparatelyScoreElements"` | Checkboxes |
| `"TrueFalse"` | key has `True`/`False` entries | Two-option radio |
| `"ShortAnswer"` | string comparison method or single string key | Text input |
| `"NumericRange"` | `"Number"` or `"Quantity"` method + Tolerance | Slider |
| `"SelectCompletion"` | explicit only | Fill-in-the-blank from choices |
| `"ClickTheAnswer"` | explicit only | Click to select answer |
| `"HotSpot"` | explicit only | Click on image |

**Explicit interface example:**
```wl
QuestionObject[
  QuestionInterface["ShortAnswer", <|"Prompt" -> "Name a prime between 10 and 20"|>],
  AssessmentFunction[{11, 13, 17, 19}, "Number"]
]
```

---

## Comparison Methods

Specify in `AssessmentFunction[key, "method"]` or inside the settings Association.

For open-ended answers like "ShortAnswer", the `ComparisonMethod` determines how the submitted answer is compared to the key. Choosing the right method is crucial for correct assessment.


| Method | Best for | Distance |
|---|---|---|
| `"Number"` | numeric answers | `Norm[#1-#2]&` |
| `"Quantity"` | answers with units (`Quantity[...]`) | `Norm[#1-#2]&` |
| `"String"` | text, case-insensitive option | `EditDistance` |
| `"Expression"` | exact WL expressions | exact match |
| `"HeldExpression"` | expressions that must not evaluate | exact match |
| `"AlgebraicValue"` | equation solving — multiple algebraically equivalent forms | exact match |
| `"ArithmeticResult"` | arithmetic computation answers | exact match |
| `"PolynomialResult"` | polynomial answers | exact match |
| `"CalculusResult"` | derivatives, integrals | exact match |
| `"CodeEquivalence"` | WL code that should behave the same | behavioral |
| `"Date"` | `DateObject` answers | `DateDifference` |
| `"GeoPosition"` | geographic locations | `GeoDistance` |
| `"Vector"` | vector answers | `Norm[#1-#2]&` |
| `"Color"` | color values | `ColorDistance` |

**With Tolerance** (for numeric-distance methods):
```wl
AssessmentFunction[9.8, <|"ComparisonMethod" -> "Number"|>, Tolerance -> 0.05]
```

**Multiple algebraically equivalent correct answers:**
```wl
AssessmentFunction[
  {-Sqrt[(1+b)/a], Sqrt[(1+b)/a]},
  "AlgebraicValue"
]
```

---

## Partial Credit and Multiple-Select (ChooseMultiple)

Use `"ListAssessment" -> "SeparatelyScoreElements"` to score each selected item independently:

```wl
QuestionObject[
  "Select all prime numbers:",
  AssessmentFunction[
    {2 -> 1, 3 -> 1, 4 -> 0, 5 -> 1, 6 -> 0},
    <|"ListAssessment" -> "SeparatelyScoreElements"|>
  ]
]
```

Other `ListAssessment` values:
- `"WholeList"` (default) — the entire submitted list is assessed as one answer
- `"AllElementsOrdered"` — list elements must match in order
- `"AllElementsOrderless"` — list elements can be in any order

---

## Explanation and LLM Feedback

Per-answer static explanation (best for discrete choice questions):
```wl
"Paris" -> <|"Score" -> 1, "Explanation" -> "Correct!"|>
```

Dynamic `ExplanationFunction` — receives an Association with keys
`"GivenAnswer"`, `"AnswerCorrect"`, `"TargetAnswer"`, `"Score"`, etc.:
```wl
AssessmentFunction[
  correctAnswer,
  <|"ExplanationFunction" -> Function[a,
    If[a["AnswerCorrect"],
      "Great work!",
      "The correct answer is " <> ToString[a["TargetAnswer"]] <> ". Try again!"
    ]
  ]|>
]
```

LLM-generated feedback using the Wolfram Prompt Repository:
```wl
AssessmentFunction[
  {"their" -> 1, "they're", "there"},
  <|"ExplanationFunction" -> LLMResourceFunction["QuestionAssessmentExplanation"]|>
]
```

---

## Custom Comparator

When built-in methods don't fit, supply your own comparison function:

```wl
(* Accept any string that contains "Newton" *)
AssessmentFunction[
  Automatic,
  <|"Comparator" -> Function[{answer, _}, StringContainsQ[answer, "Newton", IgnoreCase -> True]]|>
]
```

`"Selector"` alternative — useful when you want to pick the nearest key:
```wl
<|"Selector" -> Composition[First, Nearest]|>
```

---

## QuestionGenerator

Randomized questions — wrap in a function that returns a `QuestionObject`:
```wl
QuestionGenerator[Function[{},
  Module[{a = RandomInteger[{2, 9}], b = RandomInteger[{2, 9}]},
    QuestionObject[
      "What is " <> ToString[a] <> " × " <> ToString[b] <> "?",
      AssessmentFunction[a*b, "Number"]
    ]
  ]
]]
```

---

## AssessmentResultObject

Applying an `AssessmentFunction` directly returns an `AssessmentResultObject`:
```wl
af = AssessmentFunction[{755 -> 1, 714 -> 0, 868 -> 0}];
result = af[755]
result["Score"]          (* 1 *)
result["AnswerCorrect"]  (* True *)
result["Explanation"]    (* None, unless set *)
result["MaxScore"]       (* 1 *)
result[All]              (* full Association of all properties *)
```

---

## Common Patterns and Pitfalls

**SelectCompletion answers must be lists:**
```wl
(* Wrong *)
AssessmentFunction["red" -> 1]
(* Right — wrap in a list *)
AssessmentFunction[{{"red"} -> 1}]
```

**ChooseMultiple needs SeparatelyScoreElements to show checkboxes:**
Without it, the interface defaults to `MultipleChoice` even with multiple correct answers.

**AlgebraicValue requires `Hold` internally** — you write the expressions normally; WL wraps them automatically. Don't `Hold` them yourself.

**Tolerance only applies to distance-based methods** (`"Number"`, `"Quantity"`, `"Vector"`, `"GeoPosition"`, `"Date"`, `"Color"`). It has no effect on `"Expression"` or `"AlgebraicValue"`.

---

## Quick Decision Guide

| User wants… | Use |
|---|---|
| Pick one from a list | `AssessmentFunction[{a->1, b->0, c->0}]` |
| Pick many from a list | `AssessmentFunction[{a->1,b->0,c->1}, <\|"ListAssessment"->"SeparatelyScoreElements"\|>]` |
| True or false | `AssessmentFunction[{True->1, False->0}]` |
| Numeric answer ±tolerance | `AssessmentFunction[val, <\|"ComparisonMethod"->"Number"\|>, Tolerance->δ]` |
| Algebra / solving equations | `AssessmentFunction[{ans1, ans2}, "AlgebraicValue"]` |
| Calculus answer | `AssessmentFunction[ans, "CalculusResult"]` |
| Code that should work the same | `AssessmentFunction[codeExpr, "CodeEquivalence"]` |
| Free text | `AssessmentFunction[ans, "String"]` or custom `"Comparator"` |
| Physical quantity with units | `AssessmentFunction[Quantity[...], "Quantity"]` |
| Randomized question | `QuestionGenerator[Function[{}, ...]]` |
