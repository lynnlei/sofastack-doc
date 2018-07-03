在 Jarslink2.0 中，对合并部署的定义是指在同一个 JVM 中，加载运行多个 Biz 包。在[应用打包](jarslink-repackage.md)这一节中，我们描述了 Spring Boot/SOFABoot 应用和 Biz 包的关系。因此也可以认为，这里的合并部署是指在同一个 JVM 中，加载运行多个 Spring Boot/SOFABoot 应用。

在[应用打包](jarslink-repackage.md)这一节文末中提到 Biz 包可以通过 mvn deploy 指令发布到远程 maven 仓库，类似发布普通 Jar 包。很自然想到，这样做的好处是应用可以通过依赖的形式引入其他应用生成的 Biz 包，就像引入普通的 Jar 包依赖。那么，在应用中引入其他应用生成的 Biz 包的目的什么？还有，Jarslink2.0 怎么动态装卸载 Biz 包？

回答上面两个问题，其实就是理解静态合并部署和动态合并部署的概念。

## 静态合并部署
回答第一个问题，在应用中引入其他应用生成的 Biz 包目的是什么？

[应用打包](jarslink-repackage.md)一节中描述了如何将应用打包成 Ark 包，也给了一个粗糙的等式，Ark 包 = Biz 包 + SOFAArk 框架 + Ark Plugin。然而当应用中引入了其他应用生成的 Biz 包，打出来的 Ark 包会是怎样的呢？ 结论是打包插件会区别看待 Biz 包这种特殊的依赖包，插件会把所有的非 Biz 包依赖打包进应用的 Biz 包，但是对于引入的 Biz 包，会看成是和当前应用 Biz 包平等的关系，因此在最后打出的 Ark 包中，会包含多个 Biz 包，具体可以参考[Ark 包目录结构](https://alipay.github.io/sofastack.github.io/docs/ark-jar.html#Ark-%E5%8C%85%E5%85%B8%E5%9E%8B%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84)。此时，通过java -jar 启动该 Ark 包，会发现包含的 Biz 包都会被启动。

总结一下，即应用通过依赖的形式引入了其他应用生成的 Biz 包，最后由该应用打包生成的 Ark 包将会包含多个 Biz 包；执行该 Ark 包，将启动所有的 Biz 包，被称之为静态合并部署。

静态合并部署不依赖 Jarslink2.0，直接通过 SOFAArk 打包插件即可使用这一特性。

**请注意，多个 Biz 包的启动顺序是可控的，每个 Biz 包在生成时，可以通过打包插件配置优先级，默认值为 100，优先级越高，值越小。优先级即决定了 Biz 包的启动顺序。**

## 动态合并部署
回答第二个问题，Jarslink2.0 怎么动态装卸载 Biz 包？

相对于静态合并部署，Jarslink2.0 赋予了 SOFAArk 框架动态装卸载 Biz 包的能力。即在启动 Ark 包之后，如果 Ark 包中含有了 Jarslink2.0 插件，那么在运行时，Jarslink2.0 可以动态地接受指令控制 Biz 包的装载和卸载。Jarslink2.0 目前只接受 telnet 协议连接的方式进行指令的交互，端口为 1234。 启动 Ark 包后， 执行 `telnet ip 1234` 即可进入 Jarslink2.0 交互界面。未来计划在 0.2.0 版本中支持 Http 请求的方式进行交互。

关于 Jarslink2.0 的指令模式将在专门的[Jarslink 交互指令](jarslink-instruction.md)一节中介绍，下面我们着重描述如何使用 Jarslink2.0 实现应用的热更新。这里说的应用热更新是指，运行时使用新版本的应用替换老版本的应用，中间保证应用对外提供服务的连续性。

在合并部署时，Jarslink2.0 是通过 Biz 包名称和版本号作为唯一标识。Biz 包名称和版本号在插件打包过程中生成，记录在 MANIFEST.MF 文件中。Jarslink2.0 允许同时安装相同名称不同版本的 Biz 包，但是最多只有一个相同名称的 Biz 包对外提供服务。这里说的服务是指 Biz 包发布的 JVM 服务，关于 Biz 包 JVM 服务，请阅读[Ark Biz 通信](../) 一节。当 Jarslink2.0 已经加载两个相同名称不同版本的 Biz 包时，可以通过命令切换对外提供服务的 Biz 版本，从而做到应用热更新，并保证服务的连续性。

**请注意，不论是静态合并部署还是动态合并部署，不支持多个 Web 应用。**