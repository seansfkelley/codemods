# codemods

A collection of transforms for use with
[facebook/jscodeshift](https://github.com/facebook/jscodeshift).

<div align="center">

[![Gitter Chat for codemods](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/JamieMason/codemods)
[![Donate via PayPal](https://img.shields.io/badge/donate-paypal-blue.svg)](https://www.paypal.me/foldleft)
[![Analytics](https://ga-beacon.appspot.com/UA-45466560-5/codemods?flat&useReferer)](https://github.com/igrigorik/ga-beacon)
[![Follow JamieMason on GitHub](https://img.shields.io/github/followers/JamieMason.svg?style=social&label=Follow)](https://github.com/JamieMason)
[![Follow fold_left on Twitter](https://img.shields.io/twitter/follow/fold_left.svg?style=social&label=Follow)](https://twitter.com/fold_left)

[Installation](#installation) | [Usage](#usage) | [Contributing](#contributing) |
[Transforms](#transforms) | [Quick Intro To Making A Codemod](#quick-intro-to-making-a-codemod)

</div>

## Installation

```sh
git clone https://github.com/JamieMason/codemods.git
cd codemods
yarn install
```

## Usage

```
# yarn
yarn name-of-the-transform <path-to-file>

# npm
npm run name-of-the-transform -- <path-to-file>

# jscodeshift
jscodeshift -t ./transforms/name-of-the-transform.js <path-to-file>
```

## Contributing

Transforms can be created at `./transforms/<transform-name>.js` and tested by adding example input
files at `./test/fixtures/<transform-name>/<scenario-name>.input.js` with the corresponding expected
output alongside it at `./test/fixtures/<transform-name>/<scenario-name>.output.js`.

All fixtures are discovered and tested when running `yarn test`.

## Transforms

### import-from-root

Rewrite deep imports to import from a packages' root index.

> Set the Environment Variable `IMPORT_FROM_ROOT` to apply this transform only to packages whose
> name starts with that string: `IMPORT_FROM_ROOT=some-package yarn import-from-root <path-to-file>`

```js
/* INPUT */
import { foo } from 'some-package/foo/bar/baz';

/* OUTPUT */
import { foo } from 'some-package';
```

### sort-jsx-props

Sort props of JSX Components alphabetically.

```jsx
/* INPUT */
<Music zootWoman={true} rickJames={true} zapp={true} />

/* OUTPUT */
<Music rickJames={true} zapp={true} zootWoman={true} />
```

### sort-object-props

Sort members of Object Literals alphabetically.

```js
/* INPUT */
const players = { messi: true, bergkamp: true, ginola: true };

/* OUTPUT */
const players = { bergkamp: true, ginola: true, messi: true };
```

### use-named-exports

Naively convert a default export to a named export using the name of the file, which may clash with
other variable names in the file. This codemod would need following up on with ESLint and some
manual fixes.

```js
/* INPUT */
// ~/Dev/repo/src/apps/health/server.js
export default mount('/health', app);

/* OUTPUT */
// ~/Dev/repo/src/apps/health/server.js
export const server = mount('/health', app);
```

### use-named-imports

Naively convert a default import to a named import using the original name, which may not match what
the module is actually exporting. This codemod would need following up on with ESLint and some
manual fixes.

```js
/* INPUT */
import masthead from '@sky-uk/koa-masthead';

/* OUTPUT */
import { masthead } from '@sky-uk/koa-masthead';
```

## Quick Intro To Making A Codemod

1. Open [ASTExplorer][astexplorer] with the Parser set to `esprima` and Transform set to
   `jscodeshift`.
1. Paste some **Source** in the Top-Left Panel which you want to Transform.
1. Edit your **Codemod** in the Bottom-Left Panel.

There will be 4 Panels:

| Panel        | Purpose                                                        |
| :----------- | :------------------------------------------------------------- |
| Top-Left     | **Source** you want to transform                               |
| Top-Right    | The **AST** of your **Source**                                 |
| Bottom-Left  | Your **Codemod** Script                                        |
| Bottom-Right | The **Result** of applying your **Codemod** to your **Source** |

The [docs for jscodeshift](https://github.com/facebook/jscodeshift) aren't enough and you will need
to refer to ast-type [definitions] to know what is available. Using
[VariableDeclaration][variabledeclaration] as an example, you can find all variable declarations
using the PascalCase `j.VariableDeclaration`

```js
j(file.source).find(j.VariableDeclaration);
```

and create new variable declarations using the camelCase `j.variableDeclarator`.

```js
j.variableDeclaration('const', [j.variableDeclarator(j.identifier(fileName), exportedValue)]);
```

The [VariableDeclaration][variabledeclaration] definition shows what `fields` it takes, which are
the arguments the `j.variableDeclarator` function takes, which are an `Identifier` (a variable with
a name but no value), or a `VariableDeclarator` (a variable with a name as well as a value
assigned).

If we look up the definition of [VariableDeclarator][variabledeclarator] we see it takes two
arguments called `id` and `init`. The `id` is an identifier `j.identifier('varName')` and `init` is
a value to initialise the variable with, which should be the AST of whatever value you want to
assign. For a simple literal value, that would be `j.literal('hello')`.

Putting that all together you have a Hello World Codemod of:

```js
export default (file, api) => {
  const j = api.jscodeshift;

  // Have a look in the console at what APIs are available
  console.log({
    jscodeshiftAPI: j,
    fileAPI: j(file.source)
  });

  return j(file.source)
    .find(j.Program)
    .forEach(path => {
      // Unwrap the AST node from this wrapper
      const emptyFile = path.value;

      // add a comment
      const singleLineComment = j.commentLine(' Hello World');
      const varName = j.identifier('hello');
      emptyFile.comments = [singleLineComment];

      // add a const
      const literalString = j.literal('world');
      const nameValuePair = j.variableDeclarator(varName, literalString);
      const constVarStatement = j.variableDeclaration('const', [nameValuePair]);
      emptyFile.body = [constVarStatement];
    })
    .toSource();
};
```

Good luck!

<!-- Links -->

[astexplorer]:
  https://astexplorer.net/#/gist/47f549f753f541aff11c492c89ae82fa/e56b2df09a8e868c86139bc39ea631a0a725cbf6
[definitions]: https://github.com/benjamn/ast-types/tree/master/def
[variabledeclaration]: https://github.com/benjamn/ast-types/blob/v0.11.7/def/esprima.js#L9-L13
[variabledeclarator]:
  https://github.com/benjamn/ast-types/blob/a7eaba5ecc79a58acb469cbbf9fe7603cec9f57e/def/core.js#L190-L194
