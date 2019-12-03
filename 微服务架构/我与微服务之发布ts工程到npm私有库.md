## 一、问题背景
&emsp;&emsp;微服务开发时，前端往往会有一些公共组件，这些组件一般发布到`npm`私有库，本文将介绍如何构建`ts`工程，
并将`ts`工程发布到`npm`私有库。
## 二、构建ts工程
1. 安装nodejs、npm、tsc
```sh
apt-get install nodejs
apt-get install npm
npm install -g tsc
```
2. 构建npm项目
```sh
touch projectname
npm init
```
```sh
vim package.json
```
```json
{
  "name": "pecker-eyes",
  "version": "1.0.1",
  "description": "query filters",
  "scripts": {
    "test": "node ./build/test/test.js" //测试脚本位置 npm test执行测试
  },
  "author": "xpzsoft",
  "license": "ISC",
  "build": "tsc", // 构建脚本命令 tsc编译目录及子目录下的所有ts文件 npm build
  "release": "tsc && npm publish" // 发布命令 npm release
}
```
3. 构建ts配置文件
```sh
tsc --init
vim tsconfig.json
```

```json
{
  "compilerOptions": {
    /* Basic Options */
    "target": "es5",                          /* Specify ECMAScript target version: 'ES3' (default), 'ES5', 'ES2015', 'ES2016', 'ES2017','ES2018' or 'ESNEXT'. */
    "module": "commonjs",                     /* Specify module code generation: 'none', 'commonjs', 'amd', 'system', 'umd', 'es2015', or 'ESNext'. */
    // "lib": [],                             /* Specify library files to be included in the compilation. */
    // "allowJs": true,                       /* Allow javascript files to be compiled. */
    // "checkJs": true,                       /* Report errors in .js files. */
    // "jsx": "preserve",                     /* Specify JSX code generation: 'preserve', 'react-native', or 'react'. */
    "declaration": true,                   /* Generates corresponding '.d.ts' file. */
    // "declarationMap": true,                /* Generates a sourcemap for each corresponding '.d.ts' file. */
    // "sourceMap": true,                     /* Generates corresponding '.map' file. */
    // "outFile": "./",                       /* Concatenate and emit output to single file. */
    "outDir": "./build",                        /* Redirect output structure to the directory. */
    // "rootDir": "./",                       /* Specify the root directory of input files. Use to control the output directory structure with --outDir. */
    // "composite": true,                     /* Enable project compilation */
    // "removeComments": true,                /* Do not emit comments to output. */
    // "noEmit": true,                        /* Do not emit outputs. */
    // "importHelpers": true,                 /* Import emit helpers from 'tslib'. */
    // "downlevelIteration": true,            /* Provide full support for iterables in 'for-of', spread, and destructuring when targeting 'ES5' or 'ES3'. */
    // "isolatedModules": true,               /* Transpile each file as a separate module (similar to 'ts.transpileModule'). */

    /* Strict Type-Checking Options */
    "strict": true,                           /* Enable all strict type-checking options. */
    // "noImplicitAny": true,                 /* Raise error on expressions and declarations with an implied 'any' type. */
    // "strictNullChecks": true,              /* Enable strict null checks. */
    // "strictFunctionTypes": true,           /* Enable strict checking of function types. */
    // "strictPropertyInitialization": true,  /* Enable strict checking of property initialization in classes. */
    // "noImplicitThis": true,                /* Raise error on 'this' expressions with an implied 'any' type. */
    // "alwaysStrict": true,                  /* Parse in strict mode and emit "use strict" for each source file. */

    /* Additional Checks */
    // "noUnusedLocals": true,                /* Report errors on unused locals. */
    // "noUnusedParameters": true,            /* Report errors on unused parameters. */
    // "noImplicitReturns": true,             /* Report error when not all code paths in function return a value. */
    // "noFallthroughCasesInSwitch": true,    /* Report errors for fallthrough cases in switch statement. */

    /* Module Resolution Options */
    // "moduleResolution": "node",            /* Specify module resolution strategy: 'node' (Node.js) or 'classic' (TypeScript pre-1.6). */
    // "baseUrl": "./",                       /* Base directory to resolve non-absolute module names. */
    // "paths": {},                           /* A series of entries which re-map imports to lookup locations relative to the 'baseUrl'. */
    // "rootDirs": [],                        /* List of root folders whose combined content represents the structure of the project at runtime. */
    // "typeRoots": [],                       /* List of folders to include type definitions from. */
    // "types": [],                           /* Type declaration files to be included in compilation. */
    // "allowSyntheticDefaultImports": true,  /* Allow default imports from modules with no default export. This does not affect code emit, just typechecking. */
    "esModuleInterop": true                   /* Enables emit interoperability between CommonJS and ES Modules via creation of namespace objects for all imports. Implies 'allowSyntheticDefaultImports'. */
    // "preserveSymlinks": true,              /* Do not resolve the real path of symlinks. */

    /* Source Map Options */
    // "sourceRoot": "",                      /* Specify the location where debugger should locate TypeScript files instead of source locations. */
    // "mapRoot": "",                         /* Specify the location where debugger should locate map files instead of generated locations. */
    // "inlineSourceMap": true,               /* Emit a single file with source maps instead of having a separate file. */
    // "inlineSources": true,                 /* Emit the source alongside the sourcemaps within a single file; requires '--inlineSourceMap' or '--sourceMap' to be set. */

    /* Experimental Options */
    // "experimentalDecorators": true,        /* Enables experimental support for ES7 decorators. */
    // "emitDecoratorMetadata": true,         /* Enables experimental support for emitting type metadata for decorators. */
  }
}

```

4. 项目根目录下构建`index.js`，export对外提供的ts类型

```javascript
/*
 * @Description: 接口暴露文件
 * @Author: xpzsoft
 * @Date: 2019-11-28 16:07:52
 * @LastEditors: xpzsoft
 * @LastEditTime: 2019-11-28 16:10:58
 */
export {QueryUtils} from './filter/QueryUtils';
export {ComposeFilter} from './filter/impl/ComposeFilter';
```

## 三、发布ts项目
1. 编译项目
```sh
npm build
vim package.json
```

```json
{
  "name": "pecker-eyes",
  "version": "1.0.1",
  "description": "query filters",
  "scripts": {
    "test": "node ./build/test/test.js" //测试脚本位置 npm test执行测试
  },
  "author": "xpzsoft",
  "license": "ISC",
  "build": "tsc", // 构建脚本命令 tsc编译目录及子目录下的所有ts文件 npm build
  "release": "npm publish" // 发布命令 npm release
  "main": "./build/index.js" //对外暴露接口
  "types": "./build/index.d.ts"// 对外暴露ts接口
  "dependencies": {
    "typescript": "next",
    "browserify": "latest",
    "@types/browserify": "latest"
  } 
}
```

2. 发布项目
```sh
npm publish
```