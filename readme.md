# SPIR-V Tools

## Overview

The project includes an assembler, disassembler, and validator for SPIR-V, all
based on a common static library. The library contains all of the implementation
details and is used in the standalone tools whilst also enabling integration
into other code bases directly.

## Supported features

### Assembler and disassembler

* Based on Revision 31 of SPIR-V
* All core instructions are supported, except:
  * Changes to operand enum values and their syntax since Rev 30 will not work.
  * Image operands (section 3.14) are not supported, for example.
  * Test coverage is very limited.
* All GLSL std450 extended instructions are supported.
* Assembler only does basic syntax checking.  No cross validation of IDs or
  types is performed.
* OpenCL extended instructions are not supported.

### Validator

The validator is incomplete.  See the Future Work section for more information.

## CHANGES (for tools hackers)

2015-09-11
* Assembly format must be consistent across the entire source module.
  * Add API assembler and disassembler entry points to control the format.
  * Add command line options to assembler and disassembler to force it:
    --syntax-format=assignment
    --syntax-format=canonical
    The default is "assignment".
* Fixes decorations:
  * Names: SaturatedConversion, FuncParamAttr NoCapture
  * Values: Fixes values for some decorations: BuiltIn LocalInvocationId, and BuiltIn
    SubgroupId
  * All handling of FPFastMathMode masks.
  * LinkageAttributes now requires the literal string operand.
* Fixes capabilities: Adds ImageMipmap, and capabilities from LiteralSampler through
  SampleRateShading.

2015-09-09
* Avoid confusion about ownership of storage:
  * `spv_binary` is only used for output of the assembler, and should
  always be destroyed with `spvBinaryDestroy`.
  * `spv_text` is only used for output of the disassembler, and should
  always be destroyed with `spvTextDestroy`.
  * Inputs to the assembler and disassembler are provided as pointer
  and length arguments.
* Fixed parsing of floating point literals.
* Fixed the -p option for the disassembler executable.
* Fixed a build break on MSVC when using a ternary operator with conflicting
  types.
* More test coverage and other cleanups.

2015-09-04
* The parser has been overhauled
  * We use an automatically generated table to describe the syntax of each
    core instruction.  The changes to the SPIR-V spec document generator to
    create this table are still being developed.
  * The parser uses a dynamically updated list of expected operand types.
    It is expanded as needed for variable-length lists of operands, and
    consumed during the parse.  See the uses of `spv_operand_pattern_t`.
  * The syntax of enum operands and their potential operands is still
    hand-coded.  (That might change depending on the cost-benefit tradeoff.)
* We are actively increasing test coverage.
* We have tweaked the CMake build rules to make it easier to integrate
  into other packages.  Google is integrating SPIR-V Tools into the Vulkan
  conformance test suite.
* New code tends to use Google C++ style, including formatting as generated
  by `clang-format --style=google`.
* The spvBinaryToText and spvTextToBinary interfaces have been updated to
  remove a conceptual ambiguity that arises when cleaning up `spv_binary_t`
  and `spv_text_t` objects.


## Where is the code?

The `master` branch of the repository is maintained by
Kenneth Benzie `k.benzie@codeplay.com`.

*You are looking at the `google` branch.*  Google plans to maintain a linear
history of commits.

Please submit any merge requests as stated in these
[instructions](https://cvs.khronos.org/wiki/index.php/How_to_access_and_use_the_Khronos_Gitlab_Repository).


## Build

The project uses CMake to generate platform-specific build configurations. To
generate these build files issue the following commands.

```
mkdir <spirv-dir>/build
cd <spirv-dir>/build
cmake [-G<platform-generator>] ..
```

Once the build files have been generated, build using your preferred
development environment.

### CMake Options

* `SPIRV_USE_SANITIZER=<sanitizer>` - on UNIX platforms with an appropriate
  version of `clang` this option enables the use of the sanitizers documented
  [here](http://clang.llvm.org/docs/UsersManual.html#controlling-code-generation),
  this should only be used with a debug build, disabled by default
* `SPIRV_COLOR_TERMINAL=ON` - enables color console output, enabled by default
* `SPIRV_WARN_EVERYTHING=OFF` - on UNIX platforms enable the `-Weverything`
  compiler front end option, disabled by default
* `SPIRV_WERROR=OFF` - on UNIX platforms enable the `-Werror` compiler front end
  option, disabled by default

## Library

### Usage

In order to use the library from an application, the include path should point to
`<spirv-dir>/include`, which will enable the application to include the header
`<spirv-dir>/include/libspirv/libspirv.h` then linking against the static
library in `<spirv-build-dir>/bin/libSPIRV.a` or
`<spirv-build-dir>/bin/SPIRV.lib`. The intention is for this to be a C API,
however currently it relies on the generated header `spirv.h` meaning this is
currently a C++ API.

* `SPIRV` - the static library CMake target outputs `<spirv-dir>/lib/libSPIRV.a`
  on Linux/Mac or `<spirv-dir>/lib/SPIRV.lib` on Windows.

#### Entry Points

There are three main entry points into the library.

* `spvTextToBinary` implements the assembler functionality.
* `spvBinaryToText` implements the disassembler functionality.
* `spvValidate` implements the validator functionality.

### Source

In addition to the interface header `<spirv-dir>/include/libspirv/libspirv.h`
the implementation source files reside in `<spirv-dir>/source/*`.

The parsers for the assembler and disassembler use a table describing the
syntax of each core instruction.  This table can be generated from the SPIR-V
document generator:

1. Apply the patch in `source/core_syntax_table.patch` to the document generator.
2. Run the document generator with the `-a` option and place the results in
the `opcode.inc` file in the SPIR-V Tools `source` directory.
3. Be aware of version skew: The SPIR-V document generator might target a newer
verison of the spec than targeted by the SPIR-V tools.

### Assembler

The standalone assembler is the binary called `spirv-as` and is located in
`<spirv-build-dir>/bin/spirv-as`. The functionality of the assembler is
implemented by the `spvTextToBinary` library function.

The assembler operates on the textual form.

* `spirv-as` - the standalone assembler
  * `<spirv-dir>/bin/spirv-as`

#### Options

* `-o <filename>` is used to specify the output file, otherwise this is set to
  `out.spv`.

#### Format

The assembly attempts to adhere to the binary form as closely as possible using
text names from that specification. Here is an example.

```
OpCapability Shader
OpMemoryModel Logical Simple
OpEntryPoint GLCompute %3 "main"
OpExecutionMode %3 LocalSize 64 64 1
OpTypeVoid %1
OpTypeFunction %2 %1
OpFunction %1 %3 None %2
OpLabel %4
OpReturn
OpFunctionEnd
```

<a name="assignment-format"></a>
In order to improve the text's readability, the `<result-id>` generated by an
instruction can be moved to the beginning of that instruction and followed by
an `=` sign. This allows us to distinguish between variable defs and uses and
locate variable defs more easily. So, the above example can also be written as:

```
     OpCapability Shader
     OpMemoryModel Logical Simple
     OpEntryPoint GLCompute %3 "main"
     OpExecutionMode %3 LocalSize 64 64 1
%1 = OpTypeVoid
%2 = OpTypeFunction %1
%3 = OpFunction %1 None %2
%4 = OpLabel
     OpReturn
     OpFunctionEnd
```

Each line encapsulates one and only one instruction, or an OpCode and all of its
operands. OpCodes use the names provided in section 3.28 Instructions of the
SPIR-V specification, immediate values such as Addressing Model, Memory Model,
etc. use the names provided in sections 3.2 Source Language through 3.27
Capability of the SPIR-V specification. Literals strings are enclosed in quotes
`"<string>"` while literal numbers have no special formatting.

##### ID Definitions & Usage

An ID definition pertains to the `<result-id>` of an OpCode, and ID usage is any
input to an OpCode. All IDs are prefixed with `%`. To differentiate between
defs and uses, we suggest using the second format shown in the above example.

##### Named IDs

The assembler also supports named IDs, or virtual IDs, which greatly improves
the readability of the assembly. The same ID definition and usage prefixes
apply. Names must begin with an character in the range `[a-z|A-Z]`. The
following example will result in identical SPIR-V binary as the example above.

```
          OpCapability Shader
          OpMemoryModel Logical Simple
          OpEntryPoint GLCompute %main "main"
          OpExecutionMode %main LocalSize 64 64 1
  %void = OpTypeVoid
%fnMain = OpTypeFunction %void
  %main = OpFunction %void None %fnMain
%lbMain = OpLabel
          OpReturn
          OpFunctionEnd
```

##### Arbitrary Integers

*Warning: Not all of the following has been implemented*

When writing tests it can be useful to emit an invalid 32 bit word into the
binary stream at arbitrary positions within the assembly. To specify an
arbitrary word into the stream the prefix `!` is used, this takes the form
`!<integer>`. Here is an example.

```
OpCapability !0x0000FF00
```

Any word in a valid assembly program may be replaced by `!<integer>` -- even
words that dictate how the rest of the instruction is parsed.  Consider, for
example, the following assembly program:

```
%4 = OpConstant %1 123 456 789 OpExecutionMode %2 LocalSize 11 22 33
OpExecutionMode %3 InputLines
```

The words `OpConstant`, `LocalSize`, and `InputLines` may be replaced by random
`!<integer>` values, and the assembler will still assemble an output binary with
three instructions.  It will not necessarily be valid SPIR-V, but it will
faithfully reflect the input text.

You may wonder how the assembler recognizes the instruction structure (including
instruction boundaries) in the text with certain crucial words replaced by
arbitrary integers.  If, say, `OpConstant` becomes a `!<integer>` whose value
differs from the binary representation of `OpConstant` (remember that this
feature is intended for fine-grain control in SPIR-V testing), the assembler
generally has no idea what that value stands for.  So how does it know there is
exactly one `<id>` and three number literals following in that instruction,
before the next one begins?  And if `LocalSize` is replaced by an arbitrary
`!<integer>`, how does it know to take the next three words (instead of zero or
one, both of which are possible in the absence of certainty that `LocalSize`
provided)?  The answer is a simple rule governing the parsing of instructions
with `!<integer>` in them:

When a word in the assembly program is a `!<integer>`, that integer value is
emitted into the binary output, and parsing proceeds differently than before:
each subsequent word not recognized as an OpCode is emitted into the binary
output without any checking; when a recognizable OpCode is eventually
encountered, it begins a new instruction and parsing returns to normal.  (If a
subsequent OpCode is never found, then this alternate parsing mode handles all
the remaining words in the program.  If a subsequent OpCode is in an
[assignment format](#assignment-format), the ID preceding it begins a new
instruction, even if that ID is itself a `!<integer>`.)

The assembler processes the words encountered in alternate parsing mode as
follows:

* If the word is a number literal, it outputs that number as one or more words,
  as defined in the SPIR-V specification for Literal Number.
* If the word is a string literal, it outputs a sequence of words representing
  the string as defined in the SPIR-V specification for Literal String.
* If the word is an ID, it outputs the ID's internal number.  If no such number
  exists yet, a unique new one will be generated.  (Uniqueness is at the
  translation-unit level: no other ID in the same translation unit will have the
  same number.)
* If the word is another `!<integer>`, it outputs that integer.
* Any other word causes the assembler to quit with an error.

Note that this has some interesting consequences, including:

* When an OpCode is replaced by `!<integer>`, the integer value should encode
  the instruction's word count, as specified in the physical-layout section of
  the SPIR-V specification.

* Consecutive instructions may have their OpCode replaced by `!<integer>` and
  still produce valid SPIR-V.  For example, `!262187 %1 %2 "abc" !327739 %1 %3 6
  %2` will successfully assemble into SPIR-V declaring a constant and a
  PrivateGlobal variable.

* Enums (such as `DontInline` or `SubgroupMemory`, for instance) are not handled
  by the alternate parsing mode.  They must be replaced by `!<integer>` for
  successful assembly.

* The `=` sign cannot be processed by the alternate parsing mode if the OpCode
  following it is a `!<integer>`.

* When replacing a named ID with `!<integer>`, it is possible to generate
  unintentionally valid SPIR-V.  If the integer provided happens to equal a
  number generated for an existing named ID, it will result in a reference to
  that named ID being output.  This may be valid SPIR-V, contrary to the
  presumed intention of the writer.

### Disassembler

The standalone disassembler is the binary called `spirv-dis` and is located in
`<spirv-build-dir>/bin/spirv-dis`. The functionality of the disassembler is
implemented by the `spvBinaryToText` library function.

The disassembler operates on the binary form.

* `spirv-dis` - the standalone disassembler
  * `<spirv-dir>/bin/spirv-dis`

#### Options

* `-o <filename>` is used to specify the output file, otherwise this is set to
  `out.spvasm`.
* `-p` prints the assembly to the console on stdout, this includes colored
  output on Linux, Windows, and Mac.

### Validator

The standalone validator is the binary called `spirv-val` and is located in
`<spirv-build-dir>/bin/spirv-val`. The functionality of the validator is
implemented by the `spvValidate` library function.

The validator operates on the binary form.

* `spirv-val` - the standalone validator
  * `<spirv-dir>/bin/spirv-val`

#### Options

* `-basic` performs basic stream validation, currently not implemented.
* `-layout` performs logical layout validation as described in section 2.16
  Validation Rules, currently not implemented.
* `-id` performs ID validation according to the instruction rules in sections
  3.28.1 through 3.28.22, enabled but is a work in progress.
* `-capability` performs capability validation and or reporting, currently not
  implemented.

## Tests

The project contains a number of tests, implemented in the `UnitSPIRV`
executable, used to drive the development and correctness of the tools, these
use the [googletest](https://github.com/google/googletest) framework. The
[googletest](https://github.com/google/googletest) source is not provided with
this project, to enable the tests place the
[googletest](https://github.com/google/googletest) source in the
`<spirv-dir>/external/googletest` directory, rerun CMake if you have already
done so previously, CMake will detect the existence of
`<spirv-dir>/external/googletest` then build as normal.

## Future Work

### Assembler and disassembler

* Support OpenCL extension libraries.
* Enforce the parsing rules.
* Likely add an option to select from different assembly language syntaxes.

### Validator

* Adopt the parser strategy used by the text and binary parsers.
* Complete implementation of ID validation rules in `spirv-val`.
* Implement section 2.16 Validation Rules in `spirv-val`.
* Implement Capability validation and or report in `spirv-val`.
* Improve assembly output from `spirv-dis`.
* Improve diagnostic reports.

## Known Issues

* Float constant values are parsed incorrectly.
* Improve literal parsing in the assembler, currently only decimal integers and
  floating-point numbers are supported as literal operands and the parser is not
  contextually aware of the desired width of the operand.
* Sometimes the assembler will succeed, but the disassembler will fail to
  disassemble the result.
* Sometimes the disassembler fails to produce any output on an error.
* Many more issues.

## Licence

Copyright (c) 2015 The Khronos Group Inc.

Permission is hereby granted, free of charge, to any person obtaining a
copy of this software and/or associated documentation files (the
"Materials"), to deal in the Materials without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Materials, and to
permit persons to whom the Materials are furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be included
in all copies or substantial portions of the Materials.

MODIFICATIONS TO THIS FILE MAY MEAN IT NO LONGER ACCURATELY REFLECTS
KHRONOS STANDARDS. THE UNMODIFIED, NORMATIVE VERSIONS OF KHRONOS
SPECIFICATIONS AND HEADER INFORMATION ARE LOCATED AT
   https://www.khronos.org/registry/

THE MATERIALS ARE PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
MATERIALS OR THE USE OR OTHER DEALINGS IN THE MATERIALS.
