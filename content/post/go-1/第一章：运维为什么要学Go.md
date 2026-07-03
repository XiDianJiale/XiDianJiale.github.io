### 本章目标：
1. Go为什么适合运维和云原生场景？
2. 运维团队学习go的目标不是做“go后端开发”，而是为了有语言级直觉下的排障能力（这一点AI也很难帮你做）
3. 文章后续会围绕“写工具，读Kubernetes源码，排查问题”这个框架展开

### 1.1 Go与运维的关系
对于运维/SRE同学来说，学习go是非常值得的，go非常适合写稳定且易于部署的运维工具：
- 跨平台编译运行
- 云原生生态贴近
- 轻量并发
或者更好理解的，咱们谈谈常见的痛点来见go的优势：
1. 同时探测几百台机器，这种大规模并发任务你还在用并发支持低效python或者粗粒度控制的shell？
2. 不同机器的环境依赖复杂，服务器需要有python解释器，有编译工具链，而go的编译文件可以直接发给服务器上运行，服务器不需要有网，甚至不需要装Go

### 1.2 我们一般在哪些场景使用go？
- 巡检主机、端口、HTTP 接口，写探针或者检活服务
- 调用 Kubernetes API 或云厂商 API。
- 使用 pprof 分析 Go 服务的 CPU、内存和 goroutine 问题


### 1.3 关于Kubernetes的源码
Kubernetes 自 2014 年由 Google 正式开源，并于 2015 年作为首个种子项目捐赠给 CNCF（云原生计算基金会）进行孵化。
但是其实当我们阅读其源码的时候就发现，所谓的Controller、Informer、WorkQueue、Scheduler、kube-proxy这些听起来很复杂的系统设计，其实本质上大量核心逻辑都是几个go语言很基础的能力：goroutine、channel、context、锁、map、接口、显示错误处理、pprof。
> 举例：
![Screenshot 2026-07-02 at 21.52.53.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/Screenshot%202026-07-02%20at%2021.52.53.png)

-> 源码阅读收获key takeaway:
1. k8s不是写个for循环阻塞起batch pods，而是直接起batch go routine来创建pod
2. Waitgroup是goroutine常见的搭档：用于确保go routine都运行完毕且返回这种阻塞等待(鄙人之前就写过main函数已经退出了但是go routine还没跑完的垃圾代码)



## 1.4 谈谈12 factors与go
源自：Martin Fowler <企业应用架构模式>，参考： https://12factor.net/
![Screenshot 2026-07-02 at 21.47.24.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/Screenshot%202026-07-02%20at%2021.47.24.png)
