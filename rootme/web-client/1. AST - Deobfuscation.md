# JavaScript AST Deobfuscation — CTF Writeup

> **Category:** Reverse Engineering  
> **Difficulty:** Easy  
> **Flag:** `g00d_j0b_easy_deobfuscation`
> [view](https://www.root-me.org/?page=validation&id_challenge=3908&id_auteur=1089475&lang=en)

---

## Introduction

This challenge was a JavaScript reverse engineering and deobfuscation task — with a twist. Instead of raw JavaScript source code, the target was provided as an **Abstract Syntax Tree (AST)**. The goal was to manually parse the AST, reconstruct the logic, and recover the hidden strings.

---

## What Is an AST?

An **Abstract Syntax Tree (AST)** is a tree-shaped data structure that represents the syntactic structure of source code. Parsers generate ASTs internally when compiling or interpreting code.

Instead of storing code as plain text:

```js
let a = 5 + 3;
```

The parser converts it into structured objects:

```json
{
  "type": "VariableDeclaration",
  "declarations": [
    {
      "id": "a",
      "init": {
        "type": "BinaryExpression",
        "operator": "+",
        "left": 5,
        "right": 3
      }
    }
  ]
}
```

### Why Are ASTs Used?

ASTs are central to many tools in the JavaScript ecosystem:

| Tool / Domain         | Use of AST                          |
|-----------------------|--------------------------------------|
| Babel                 | Transpilation (ES6+ → ES5)           |
| ESLint                | Static code analysis & linting       |
| Minifiers / Uglifiers | Code size reduction                  |
| Compilers             | Code generation & optimization       |
| **Malware / CTFs**    | **Obfuscation to hinder analysis**   |

ASTs allow tools to analyze code structure, transform logic automatically, detect vulnerabilities, and — critically for this challenge — obfuscate intent.

> In CTF and malware analysis contexts, attackers sometimes distribute *only* the AST output to make static analysis significantly harder.

---

## Challenge Analysis

The provided file is a JavaScript AST object. At the bottom of the AST, we can identify the entry point:

```js
let sensor = gen_sensor()
console.log(sensor)
```

**Target:** Reverse-engineer `gen_sensor()` to recover the hidden value.

---

## Part 1 — Hidden String (Warm-Up)

The challenge opens with an **Immediately Invoked Function Expression (IIFE)**:

```js
(() => {
    let d = [1856, 1824, 1776, 1728, 1776, 1728, 1776]
    d = d.map(c => String.fromCharCode(c >> 4))
    console.log(d)
})()
```

### Understanding the Logic

Each integer is **right-shifted by 4 bits** (`>> 4`), then converted to its ASCII character:

| Value  | `>> 4` | ASCII | Char |
|--------|--------|-------|------|
| `1856` | `116`  | 116   | `t`  |
| `1824` | `114`  | 114   | `r`  |
| `1776` | `111`  | 111   | `o`  |
| `1728` | `108`  | 108   | `l`  |
| `1776` | `111`  | 111   | `o`  |
| `1728` | `108`  | 108   | `l`  |
| `1776` | `111`  | 111   | `o`  |

**Result:** `trololo`

---

## Part 2 — Reversing `gen_sensor()`

### Step 1 — Key Generation via Array Concatenation

Inside `gen_sensor()`, the key is built as:

```js
let sens = [10] + [45] + [65] + [78] + [47]
```

In JavaScript, adding arrays coerces them to strings, effectively joining their elements:

```
"10" + "45" + "65" + "78" + "47" = "1045657847"
```

Parsed as a number: **`1045657847`**

### Step 2 — Bit Shift to Derive XOR Key

```js
sens >>= 4
```

This right-shifts the value by 4 bits:

```
1045657847 >> 4 = 65353615
```

**XOR key:** `65353615`

---

## Part 3 — Decoding the Sensor Array

The AST encodes a large array of integers. Each element is decoded with:

```js
String.fromCharCode(c ^ sens)
```

### Example Decryption

```
65353704 ^ 65353615 = 103
ASCII(103) = 'g'
```

Applying this XOR operation across every element in the array reconstructs the hidden message character by character.

---

## 🏁 Flag

```
g00d_j0b_easy_deobfuscation
```

---

## Tools & Concepts Used

- **Bit manipulation:** right-shift (`>>`) and XOR (`^`) operators
- **JavaScript type coercion:** array-to-string implicit conversion
- **ASCII / `String.fromCharCode()`:** character decoding
- **AST analysis:** manual traversal and logic reconstruction

---

## Further Reading

- [AST Explorer](https://astexplorer.net/) — Visualize and explore ASTs interactively
- [Babel Plugin Handbook](https://github.com/jamiebuilds/babel-handbook) — Deep dive into AST transformations
- [javascript-obfuscator](https://github.com/javascript-obfuscator/javascript-obfuscator) — Common obfuscation techniques
- [MDN: Bitwise Operators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Bitwise_AND)
