#### remix 本地环境搭建

作者：李康

由于一些原因，我们可能不能很顺畅地访问 remix IDE 的[在线版本](https://remix.ethereum.org/)，因此我们需要搭建 remix 的本地版，代码仓库在 https://github.com/ethereum/browser-solidity .

我们下载 gh-pages 分支，然后修改 build/app.js 文件，找到调用 loadVersion 函数的位置，此函数用于自动加载选择好的 solidity 编译器版本，我们避免在线加载 github 上的 solidity 编译器，因此将 loadVersion 函数的第一个参数改为 "buildin"，意味着加载本地下载好的 solidity 编译器。

我们从 https://github.com/ethereum/browser-solidity 处下载的代码仓库包含 soljson.js，但是加载该文件会出现错误（原因不详），因此去 https://ethereum.github.io/solc-bin/bin/ 处下载想要的版本的编译器，然后将其重命名为 soljson.js，覆盖原先的文件。此时直接访问 index.html 就可以访问本地 remix 编译环境了
