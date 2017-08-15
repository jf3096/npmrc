# npmrc

> 分析 npm 存在的问题，提出解决方案，统一使用 npmrc 文件规范 npm

NPM 是随同 NodeJS 一起安装的包管理工具，能解决 NodeJS 代码部署上的很多问题，常见的使用场景有以下几种：

    1. 允许用户从NPM服务器下载别人编写的第三方包到本地使用。
    2. 允许用户从NPM服务器下载并安装别人编写的命令行程序到本地使用。
    3. 允许用户将自己编写的包或命令行程序上传到NPM服务器供别人使用。

但在使用的过程中我们一直被深埋坑中。

## npm 存在的各种问题

### npm 2 无法复用模块导致
npm 2 由于文件夹依赖设计并没有采取任何扁平化的策略，这样导致所有安装的包没有任何复用能力。
对于 window 的小伙伴也是一个暴击，由于没有扁平化，每一个包之间的依赖关系很容易就会嵌套很深而导致路径超过 window 上限路径240个字符从而报错。
也因此，node_modules无法通过正常的手段复制或者删除。当时解决方式采用了 rimraf 的包递归逐层删除。

**参考资料：[难道只有我一个人想吐槽npm这种包管理方式么](http://www.cnblogs.com/liuxianan/p/shit-npm.html)**


### npm3 重复和去重
npm 3 对此做了相应的优化，根据npm 3的安装机制，同版本的包会被扁平化到 node_modules 的根节点上。但是，相同包不同版本依旧留在各自被依赖来源里，
这样做并没有从根本上共享相同的的包，安装的每一个包的顺序也决定着最终 node_modules 的结构。在开发过程中，一个一个包的安装会因为没有得到真正的共享而逐渐
导致node_modules越来越大。优化方案可以删除node_modules后重新安装，或者 npm dedupe 实现去重的效果。
目前截至2017年08月15日，node官方稳定版 6.11.2 默认 npm 版本是 npm 3.10.10。

**参考资料：[NPM 3 重复和去重](http://www.jianshu.com/p/a8b971df9a49)**


### npm 4
主要对细节如搜索功能做了修正。 npm 4也对 prepublish 做了修正，把功能明确后提供命令 prepublishOnly 和 prepare。
其主要原因 prepublish 在 npm install 的时候也被执行了，语义上也让你怀疑人生。这一点在社区上也被诟病许久。
开发者通过语义往往非常容易把 prepublish 理解成 publish 的前置钩子 script。但是因为这样的设定在使用者首次执行 npm install 安装命令的时候由于执行了 prepublish 而发生报错，
在持续集成CI中 prepublish 也“意外的”被执行了2次。npm 也在期间禁止了直接 unpublish 整个包，改为只允许 unpublish 单个版本，估计主要原因在于 [Azer Koçulu 删除了自己的所有 npm 库](https://www.zhihu.com/question/41694868) 的事件。

**参考资料：[NPM 4 seeks to fix JavaScript package search](http://www.infoworld.com/article/3134051/javascript/npm-4-seeks-to-fix-javascript-package-search.html)**

### yarn
Yarn 是一个新的包管理器，用于替代现有的 npm 客户端或者其他兼容 npm 仓库的包管理工具。Yarn 保留了现有工作流的特性，优点是更快、更安全、更可靠。
它的出现解决了很多一直存在的问题：
    
    1. 依赖包通常是带版本号，在yarn之前npm4包含4以下默认允许升级。这样依赖包所属的开发者一旦没有遵循 semver 规范就会有潜在出错风险。所以yarn带来了lock。
    2. 在npm 3中也提到过，由于共享包受限于安装顺序，这样即使相同配置 node_modules 目录的结构也可能会发生变化。这种差异可能会导致类似“我的机子上可以运行，别的机子不行”的情况，并且通常要花费大量时间定位与解决。

Yarn 通过 lockfiles 文件以及一个确定性的、可靠的安装算法，解决了版本问题和 npm 的不确定性问题。Lockfile 文件把安装的软件包版本锁定在某个特定版本，并保证 node_modules 目录在所有机器上的安装结果都是相同的。Lockfile 还使用简洁的有序键名的格式，保证了每次的文件变化最小化，进行代码审查也更为简单。

Yarn 支持并行，而npm 4以下坑到没朋友。在 npm 中这些任务是按包的顺序一个个执行，这意味着必须等待上一个包被完整安装才会进入下一个；Yarn 则并行的执行这些任务，提高了性能。

为了比较，在没有使用 shrinkwrap/yarn.lock 的方式以及清理了缓存下使用 npm （5 以下） 与 Yarn 安装 express，总共安装了 42 个依赖。
npm: 9 s
Yarn: 1.37 s

Yarn 也对所有下载的包保存到yarn全局缓存目录中，在新项目安装时，yarn校验完线上版本后，Yarn 会查找全局的缓存目录，检查所需的软件包是否已被下载。如果没有，Yarn 会抓取对应的压缩包，并放置在全局的缓存目录中，因此 Yarn 支持离线安装，同一个安装包不需要下载多次。依赖也可以通过 tarball 的压缩形式放置在源码控制系统中，以支持完整的离线安装。

**参考资料：[Yarn vs npm: 你需要知道的一切](http://web.jobbole.com/88459/)
           [Facebook 新推 Yarn，或取代 npm 客户端](http://www.oschina.net/news/78072/yarn-a-new-package-manager-for-javascript)**

### npm 5
npm 发布了 5.0 版本，提供了自动记录依赖树，下载使用强校验，重写缓存系统等功能升级和改造。做了非常大的变更和优化：

1. 默认新增 package-lock.json 来记录依赖树信息，进行依赖锁定，并使用新的 shrinkwrap 格式。
2. **--save 变成了默认参数，执行 install 依赖包时默认都会带上，除非加上 --no-save。**
3. Git 依赖优化：支持指定 semver 版本安装；含有 prepare 脚本时将安装其 devDependencies 并执行脚本。
4. 使用本地目录文件作为 file 类型依赖安装时，使用创建 symlink 的方式替代原来的文件拷贝方式，提升速度。
5. 脚本更改：在 npm pack, npm publish 时新增 prepack 和 postpack 脚本；preinstall 脚本运行优先级提升到最前，并且可以修改 node_modules。
6. 包发布将同时生成 sha512 和 sha1 校验码，下载依赖时将使用强校验算法。
7. 重写整个缓存系统和 npm cache 系列命令。废除 --cache-min --cache-max 等命令，缓存系统将由 npm 自身维护，无需用户介入。
8. registry 策略调整：配置优先级高于锁文件中记录的优先级；除非使用不同 scope 的包，不再支持不同的包使用不同的 registry。

除此之外还包含一些细节优化：

1. 离线安装时将不再尝试连接网络，而是降级尝试从缓存中读取，或直接失败。
2. 锁文件（package-lock.json, npm-shrinkwrap.json）将包含 optionalDependencies。
3. 本地包（local tarball）具有 .tar, .tar.gz, 或 .tgz 后缀时才会被安装。
4. 新增 notice 为默认loglevel。
5. node-gyp 在 Windows 提供对 node-gyp.cmd 的支持。
6. 移除 ./cli.js，使用 ./bin/npm-cli.js 代替。

在上述变更中不难发现，npm 5参考了很多yarn的特性，也解决了以上提到了众多痛点。
在安装速度上yarn 的速度在大部分正常场景下还是略高一筹，不过相比之下 npm5 的差距已经很小。

![npm5-vs-yarn](/git-img/npm5-vs-yarn.png)

**参考资料：[Here’s what you need to know about npm 5](https://blog.pusher.com/what-you-need-know-npm-5/)**


## 缓存
**其实至此，不难得出一个简单的结论，请尽快升级到npm 5或者使用yarn进行版本管理。**

在我们的日常开发过程中，我们经常会徘徊在 github 和 npm 的社区上，遇到感兴趣的、合适的库我们也会手痒痒忍不住 npm install 尝试一波。npm 包的安装真的占据我们很多时间。
随着团队项目开发的过程中，项目依赖包也随着时间组件增多，在某项目的脚手架开发中也发现，部署该项目后再次安装项目花费了近10分钟，
而部分使用者在安装过程中也出现超时等问题。为了解决这个问题，也为了让我们内部包有更好的管理方式，我们开始搭建私有 npm 镜像源服务器。

![npm5-vs-yarn](/git-img/cache-structure.png)

上图简单不表明了一个简单的缓存策略，其原理很简单，与DNS域名解析原理大同小异。

举个例子，假设我的带宽是 10 Mbps。由于npm需要翻墙的原因，导致我们除了难以下载npm的包以外，其下载速度依旧非常慢，下载速度可能只能达到 **200 KB/s**，在此我们使用的是**代理下载**。
但当我们在国内环境中，我们使用淘宝 npm 镜像源，在资源丰富网络良好情况下，我们下载速度能达到当前宽带理峰值 **1.25MB/s**，在此我们使用的是**国内网络**。
但在内网中，我们使用内网网络可以达到路由器最大宽带，大多数情况下该读写不会超过硬盘读写速度，我们轻易的能达到达到**几十 MB/s**。由于公司npm 服务器不一定所属同一个局域网中，所以架构图单独分离出来，在此我们使用的是**内网网络**。
如果我们下载直接命中本机缓存，那么我们可以简单认为速度能达到硬盘无序读写的上限，非固态硬盘轻易达到 **30-50MB/s**，固态硬盘上百M不是问题(但npm创建文件结构也占据相应时间，这里不做讨论)。在此我们使用的是**本机IO读写**

所以基于**就近原则**，为了提高我们网络的下载速率，我们应该根据自己所属网络环境搭建属于自己、团队的镜像源。
但这也并不无缺点，在首次无缓存的情况下，由于本地无缓存，则会一层一层向上回溯直到 npm 镜像源拉去。拉取后会在每一个节点缓存相应安装包。
但这个开销往往不需要针对性考虑太多，首先taobao的数据源并不是直接缓存的方式，而是每 10 分钟同步一次数据源。在者新增几个节点（每经过一个服务器认作一个节点）的开销依旧能接受，毕竟节点之间网络环境会在局域网和国内网络环境中，
TTL 和 每一个服务器的缓存处理时间并不会特别大，这里可以简单的类比当前电脑直连淘宝镜像源拉去包的时间即可。但随着时间推移，缓存带来的优势会越发明显。
在日常开发中，由于前端团队使用的技术近似，那么某种意义来说，局域网的 npm 服务器就能充当一个缓存共享服务器，一次拉取，局域网缓存，方便你我他。

## 离线缓存
在npm 5 中 **--cached-min** 和 **--cached-max** 被 **--prefer-online** 和 **--prefer-offline** 取代。

--prefer-offline: 本地有缓存用缓存，无缓存，网络拉取。
--prefer-online: 优先网络拉取，无网络时使用本地缓存。

假设我们需要下载express的包，在无本地缓存的情况下无区别，那么我们探讨一下在缓存下使用--prefer-offline， --prefer-online 和默认 npm install 的区别。

npm install express

npm install express --prefer-online

npm install express --prefer-offline


![installation](/git-img/installation.png)

default（默认)：1.868s

--prefer-online：2.343s

--prefer-offline：1.112s

在多次分析中也能发现，在默认安装下速度居中，通过npm 参数 --loglevel http 可以分析出：

default（默认)：发起请求获取服务器版本checksum后对比本地缓存版本，如果 checksum 一致说明本地缓存与线上版本一致，使用本地版本。这个过程一般会返回 http status 304，在无网络下直接使用本地缓存 http status 200 <from cache>。

--prefer-online：优先线上版本，直接发起请求下载线上版本，不对本地缓存做任何校验工作，但线上版本的其他依赖包可以使用缓存。这个过程一般会返回 http status 200 + 若干 304， 在无网络下会报错。

--prefer-offline：直接使用缓存版本，不对线上版本做任何校验工作。无论在有无网络的情况下都直接使用本地缓存，这个过程一般会返回 http status 200 <from cache>。

我们不难看出最稳妥的方式是比对线上版本后确定是否需要更新缓存。

--prefer-offline 的下载速度收益巨大，主要体现于在下载一个依赖庞大的包时尤其明显。在下载某脚手架时，直接 npm install 大约 **120 秒**，而 --prefer-offline 仅仅只要 **28 秒**。

--prefer-offline 带来的收益是明显的但也有一定的不适应性。假设在首次缓存一个模块 X 后， 模块 X 的作者很快的 unpublish 了该版本，修正 bug 后迅速推送更新，那么在相同版本号下就会有2份不同的checksum，那么使用 --prefer-offline 后无法得知该变更。

但是，根据 semver 规范，并且 npm 官方对 unpublish 也有相应警告的情况下，这种发生的概率较低。因为 unpublish 无论如何也会导致我们使用的版本库不稳定。

在知名框架工具下由于他们会遵循良好的版本规范， 所以发生概率极低。其次在绝大多数情况下，同一个版本下的包我们会使用 lock file 或者 save-exact 来确保版本唯一稳定，那么我们理解成当前下载
的版本是固定唯一的，我们也希望项目其他成员也下载到同样的代码。 基于这个逻辑，同一个版本的包应该对应同一份代码，对应同一份cache，尽管那一份cache可能存在问题。而 unpublish 的行为无疑让我们间接升级了我们的小版本，
无论是否使用 cache 都会更容易产生错误，所以这个锅不在于是否使用 cahce 而是 unpublish 这个包的作者。

发现第三方包被unpublish的成本其实并不一定很高，lock file 在再次安装时发现同一个版本 checksum 不一致时会报 EINTEGRITY 的错误，通过这个错误我们轻易发现 cache 的版本和线上的并不一致。
而这一个步骤我们可以在持续集成阶段判定即可。

那么，在成吨的收益面前，我们值得花一段时间测试 --prefer-offline 的适用性，建议在安装过程中开启 verbose 或者 http 安装日志对这个过程有一个更清晰的理解和监控。


## .npmrc
为了让统一团队在 npm 下的安装使用，我们建议团队统一使用 yarn 或者 npm 5+，并可以在脚手架生成时携带 .npmrc 配置文件统一规范。

```bash
registry=http://npm-registry.seasungame.com/ # 基于项目配置npm 镜像源，这样做的好处就不需要依赖 cnpm 或者 nrm 改变全局镜像源了

save=true # 对于 npm 5 一下默认无参数安装时默认 --save

save-exact=true # 统一版本不会随着minor版本升级而变更。如果需要，请统一对某一个包进行update、upgrade操作。

loglevel=http # 日志级别使用 http， 这样我们就能知道哪些包安装比较慢，是否使用了缓存，是否需要翻墙才能解决。

prefer-offline=true # 统一使用优先离线模式，评估收益，检验是否带来风险。
```

## License

MIT © Ailun She