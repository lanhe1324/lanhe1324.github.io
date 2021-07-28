## 关于DbContext手动`Dispose`

- 对于EF，是否需要手动Dispose DbContext对象？
网上的观点有两种:
   1. 需要手动Dispose
   2. 不需要手动Dispose

> [!TIP]
> 本项目不需要手动Dispose;   
> 使用时需要了解`DbContext`的`作用域`和`生命周期`，就可以避免很多入坑的问题产生。     
> 尽管它确实实现了 IDisposable，但它只是实现了它，因此您可以在某些特殊情况下调用 Dispose 作为保护措施。默认情况下，DbContext 会自动为您管理连接。  

