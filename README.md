# lerna源码
## 准备
* 首先把源码下载到本地
* 安装依赖，这里使用 pnpm
  * lerna 使用了workspace，所以需要配置`pnpm-workspace`
  * 安装依赖,执行 `pnpm install`
* 找到入口文件，lerna的入口文件在`core/lerna/cli.js`

`pnpm-workspace.yml` 文件配置
```yml
packages:
  - 'commands/*'
  - 'core/*'
  - 'utils/*'
```
## lerna初始化过程
`core/lerna/cli.js`文件中引入了`core/lerna/index.js`
```js
#!/usr/bin/env node

"use strict";

/* eslint-disable import/no-dynamic-require, global-require */
const importLocal = require("import-local");

if (importLocal(__filename)) {
  require("npmlog").info("cli", "using local version of lerna");
} else {
  require(".")(process.argv.slice(2));
}
```
`core/lerna/index.js`
```js
"use strict";

const cli = require("@lerna/cli");

const addCmd = require("@lerna/add/command");
const bootstrapCmd = require("@lerna/bootstrap/command");
const changedCmd = require("@lerna/changed/command");
const cleanCmd = require("@lerna/clean/command");
const createCmd = require("@lerna/create/command");
const diffCmd = require("@lerna/diff/command");
const execCmd = require("@lerna/exec/command");
const importCmd = require("@lerna/import/command");
const infoCmd = require("@lerna/info/command");
const initCmd = require("@lerna/init/command");
const linkCmd = require("@lerna/link/command");
const listCmd = require("@lerna/list/command");
const publishCmd = require("@lerna/publish/command");
const runCmd = require("@lerna/run/command");
const versionCmd = require("@lerna/version/command");

const pkg = require("./package.json");

module.exports = main;

function main(argv) {
  const context = {
    lernaVersion: pkg.version,
  };

  return cli()
    .command(addCmd)
    .command(bootstrapCmd)
    .command(changedCmd)
    .command(cleanCmd)
    .command(createCmd)
    .command(diffCmd)
    .command(execCmd)
    .command(importCmd)
    .command(infoCmd)
    .command(initCmd)
    .command(linkCmd)
    .command(listCmd)
    .command(publishCmd)
    .command(runCmd)
    .command(versionCmd)
    .parse(argv, context);
}
```
通过调试执行，最终会进入上面的`main`函数中

点击cli跳转到'core/cli.js'文件中
```js
function lernaCLI(argv, cwd) {
  const cli = yargs(argv, cwd);

  return globalOptions(cli)
    .usage("Usage: $0 <command> [options]")
    .demandCommand(1, "A command is required. Pass --help to see all available commands and options.")
    .recommendCommands()
    .strict()
    .fail((msg, err) => {
      // certain yargs validations throw strings :P
      const actual = err || new Error(msg);

      // ValidationErrors are already logged, as are package errors
      if (actual.name !== "ValidationError" && !actual.pkg) {
        // the recommendCommands() message is too terse
        if (/Did you mean/.test(actual.message)) {
          log.error("lerna", `Unknown command "${cli.parsed.argv._[0]}"`);
        }

        log.error("lerna", actual.message);
      }

      // exit non-zero so the CLI can be usefully chained
      cli.exit(actual.exitCode > 0 ? actual.exitCode : 1, actual);
    })
    .alias("h", "help")
    .alias("v", "version")
    .wrap(cli.terminalWidth()).epilogue(dedent`
      When a command fails, all logs are written to lerna-debug.log in the current working directory.

      For more information, find our manual at https://github.com/lerna/lerna
    `);
}
```
我们看到使用了 [yargs](https://github.com/yargs/yargs)
## lerna解决依赖问题
在`core/lerna/package.json`，我们可以看到lerna的本地依赖是这样写的
```json
{
...
  "dependencies": {
    "@lerna/add": "file:../../commands/add",
    "@lerna/bootstrap": "file:../../commands/bootstrap",
    "@lerna/changed": "file:../../commands/changed",
    "@lerna/clean": "file:../../commands/clean",
    "@lerna/cli": "file:../cli",
    "@lerna/create": "file:../../commands/create",
    "@lerna/diff": "file:../../commands/diff",
    "@lerna/exec": "file:../../commands/exec",
    "@lerna/import": "file:../../commands/import",
    "@lerna/info": "file:../../commands/info",
    "@lerna/init": "file:../../commands/init",
    "@lerna/link": "file:../../commands/link",
    "@lerna/list": "file:../../commands/list",
    "@lerna/publish": "file:../../commands/publish",
    "@lerna/run": "file:../../commands/run",
    "@lerna/version": "file:../../commands/version",
    "import-local": "^3.0.2",
    "npmlog": "^4.1.2"
  }
}
```
这样写的好处是不用到处执行 `npm link` 了，也节省了空间

在发布时，lerna就会把本地链接解析成线上的链接，这样就解决了线上使用问题
```js
  resolveLocalDependencyLinks() {
    // resolve relative file: links to their actual version range
    const updatesWithLocalLinks = this.updates.filter((node) =>
      Array.from(node.localDependencies.values()).some((resolved) => resolved.type === "directory")
    );

    return pMap(updatesWithLocalLinks, (node) => {
      for (const [depName, resolved] of node.localDependencies) {
        // regardless of where the version comes from, we can't publish "file:../sibling-pkg" specs
        const depVersion = this.updatesVersions.get(depName) || this.packageGraph.get(depName).pkg.version;

        // it no longer matters if we mutate the shared Package instance
        node.pkg.updateLocalDependency(resolved, depVersion, this.savePrefix);
      }

      // writing changes to disk handled in serializeChanges()
    });
  }
```
## yargs的使用
在看lerna源码时，我们看到使用了yargs,所以需要先了解下yargs的使用

lerna中使用了 [yargs](https://github.com/yargs/yargs)

首先创建一个脚手架项目，在package文件中配置脚手架的命令和入口文件
```json
{
  "bin": {
    "dev": "index.js"
  }
}
```
在入口文件，先写下下面代码
```js
#!/usr/bin/env node

const yargs = require('yargs/yargs')
const { hideBin } = require('yargs/helpers')

const argv = hideBin(process.argv)

yargs(argv)
.argv
```
在控制台输入
```bash
dev --help
```
可以看到控制台会输出
```bash
选项：
  --help     显示帮助信息                                                 [布尔]
  --version  显示版本号                                                   [布尔]
```
我们还可以输入
```bash
dev --version 
```
我们可以看到yargs会帮我们默认生成两个命令
```js
yargs(argv)
.usage("Usage: <command> [options]")
.argv
```
执行--help后
```diff
+Usage: <command> [options]

选项：
  --help     显示帮助信息                                                 [布尔]
  --version  显示版本号                                                   [布尔]
```
```js
yargs.demandCommand(1, "A command is required. Pass --help to see all available commands and options.")
```
输入dev，控制台就会输出下面内容。当我们输入没有注册的命令时就会有下面提示
```
Usage: <command> [options]

选项：
  --help     显示帮助信息                                                 [布尔]
  --version  显示版本号                                                   [布尔]

A command is required. Pass --help to see all available commands and options.
```
alias用来设置别名，例如可以将`dev -v`,就相当于输入了'dev --version'
```js
yargs
.alias('h', 'help')
.alias('v', 'version')
```
wrap可以设置控制台的快读
```js
const cli = yargs(argv)

//wrap设置为 cli.terminalWidth()，可以使控制台输出的信息充满控制台
cli
.usage("Usage: <command> [options]")
.demandCommand(1, "A command is required. Pass --help to see all available commands and options.")
.alias('h', 'help')
.alias('v', 'version')
.wrap(cli.terminalWidth())
.argv
```
epilogue用来在控制台末尾输出信息
```js
const dedent = require('dedent')

cli
.usage("Usage: <command> [options]")
.demandCommand(1, "A command is required. Pass --help to see all available commands and options.")
.alias('h', 'help')
.alias('v', 'version')
.wrap(cli.terminalWidth())
.epilogue(dedent`aaa
  aaa
`)
.argv
// 不使用dedent输出
// aaa
//   aaa
// 使用dedent输出
// aaa
// aaa
```
这里使用dedent是为了去除换行前的控制

options用来定义选项
```js
yargs.options({
    debug:{
        type: 'boolean',
        describe: 'Boostrap debug mode',
        alias: 'd'
    }
})
```
执行 `dev -h` 控制台会输出
```bash
选项：
  -d, --debug    Boostrap debug mode                                                                                                     [布尔]
  -h, --help     显示帮助信息                                                                                                            [布尔]
  -v, --version  显示版本号                                                                                                              [布尔]

```

也可以使用option来替代
```js
yargs
.option('debug', {
    type: 'boolean',
    describe: 'Boostrap debug mode',
    alias: 'd'
})
```
group用来把option进行分组
```js
yargs
.group(['d'], 'dev')
```
```
dev
  -d, --debug  Boostrap debug mode                                                                                                       [布尔]

选项：
  -h, --help     显示帮助信息                                                                                                            [布尔]
  -v, --version  显示版本号   
```
command用来注册命令
```js
yargs
  .command('init [name]', 'Do init a project', yargs => {
    // init命令的选项 builder
        yargs.option('name', {
            type: 'string',
            describe: 'Name of project',
            alias: 'n'
        })
    }, argv => {
      // 用来处理该命令的操作 handler
        console.log(argv)
    })
```
command的第二种写法
```js
  .command({
        command: 'list',
        aliases: ['ls', 'la', 'll'],
        describe: 'List local package',
        builder: yargs=>{

        },
        handler: argv =>{
            console.log(argv)
        }
    })
```
recommendCommands，当我们输错命令时，添加recommendCommands就会帮我们查找最接近输入命令的命令提示给我们
```js
yargs.recommendCommands()
```
例如当我们输入一个不存在的命令dev ini 
```js
   
Usage: <command> [options]

命令：
  index.js init [name]  Do init a project
  index.js list         List local package                                                                                [aliases: ls, la, ll]

dev
  -d, --debug  Boostrap debug mode                                                                                                       [布尔]

选项：
  -h, --help     显示帮助信息                                                                                                            [布尔]
  -v, --version  显示版本号                                                                                                              [布尔]

aaa
aaa

是指 init?
```
fail用来处理失败信息
```js
    yargs.fail(err=>{
        console.log(err)
    })
```
```bash
# 输入
 dev ini
#  输出
是指 init?
```
parse用来把参数进行解析合并,
```js
const context = {
    devVersion: '1.0.0'
}
yargs
// 会把process.argv和context进行合并
  .parse(process.argv.slice(2), context)
```
例如当我们输入`dev ls`，解析后的argv
```
{
  _: [ 'ls' ],
  devVersion: '1.0.0',
  '$0': 'D:\\tools\\nodejs\\node_modules\\@dev-cli\\core\\lib\\index.js'
}
```