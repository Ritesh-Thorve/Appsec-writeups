# JavaScript AST Deobfuscation Writeup

## Introduction

This challenge was a simple JavaScript reverse engineering and deobfuscation task using an **AST (Abstract Syntax Tree)** instead of normal JavaScript source code.

The goal was to analyze the AST manually, understand how the code works, and recover the hidden strings.

## What is an AST?

An **Abstract Syntax Tree (AST)** is a tree representation of source code.

Instead of storing code as plain text:

```javascript
let a = 5 + 3;
the parser converts it into structured objects like:

json
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
Why ASTs Are Used
ASTs are heavily used in:

JavaScript parsers

Babel

ESLint

Minifiers/Uglifiers

Compilers

Malware obfuscation

Reverse engineering challenges

ASTs allow tools to:

Analyze code structure

Transform code automatically

Obfuscate logic

Detect vulnerabilities

Optimize programs

In CTFs and malware analysis, attackers sometimes provide only AST output to make reverse engineering harder.

Challenge Analysis
The file contains a JavaScript AST object. At the bottom we can see:

javascript
let sensor = gen_sensor()
console.log(sensor)
So our target is the gen_sensor() function.

Part 1 — Hidden String
The challenge starts with an immediately invoked function:

javascript
(() => {
    let d = [1856,1824,1776,1728,1776,1728,1776]
    d = d.map(c => String.fromCharCode(c >> 4))
    console.log(d)
})()
Understanding the Logic
Each number is shifted right by 4 bits:

1856 >> 4 = 116 (ASCII 116 = t)

Doing this for all values:

Value	Shifted	ASCII
1856	116	t
1824	114	r
1776	111	o
1728	108	l
1776	111	o
1728	108	l
1776	111	o
Final output: trololo

Part 2 — The gen_sensor() Function
Step 1 — Generating the Key
Inside gen_sensor():

javascript
let sens = [10]+[45]+[65]+[78]+[47]
JavaScript converts arrays into strings during concatenation:

"10" + "45" + "65" + "78" + "47"

Result: 1045657847

Step 2 — Bit Shift Operation
Next:

javascript
sens >>= 4
This means: 1045657847 >> 4

Result: 65353615 (This becomes the XOR key)

Part 3 — Decoding the Sensor Array
The challenge contains a large integer array:

javascript
[
  65353704,
  65353663,
  // ... more values
]
Each number is decoded using:

javascript
String.fromCharCode(c ^ sens)
where sens = 65353615

XOR Decryption
Example:

65353704 ^ 65353615 = 103

ASCII 103 = g

Repeating this for every element reveals the hidden message.

Final Decoded String
text
g00d_j0b_easy_deobfuscation
