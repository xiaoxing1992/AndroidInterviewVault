---
tags: [android, 面试, jetpack, paging, A]
difficulty: A
frequency: medium
---

# Paging3 分页加载架构

## 问题

> Paging3 的三层架构是什么？PagingSource 和 RemoteMediator 分别解决什么问题？如何处理网络+数据库的分页场景？

## 核心答案

Paging3 分为三层：Repository 层（PagingSource/RemoteMediator 提供数据）、ViewModel 层（Pager 创建 PagingData Flow）、UI 层（PagingDataAdapter 消费数据）。PagingSource 处理单一数据源分页，RemoteMediator 处理网络+本地缓存的多层数据源场景，实现离线可用的分页列表。

## 深入解析

### 原理层

**三层架构：**
```
UI 层:         PagingDataAdapter → LazyPagingItems (Compose)
                    ↑ collect PagingData
ViewModel 层:  Pager(...).flow → PagingData<T>
                    ↑ 
Repository 层: PagingSource<Key, Value>  — 单数据源
               RemoteMediator<Key, Value> — 网络+数据库
```

**PagingSource 工作流程：**
```kotlin
class ArticlePagingSource(private val api: ArticleApi) : PagingSource<Int, Article>() {
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Article> {
        val page = params.key ?: 1
        return try {
            val response = api.getArticles(page, params.loadSize)
            LoadResult.Page(
                data = response.articles,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.articles.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
    
    override fun getRefreshKey(state: PagingState<Int, Article>): Int? {
        return state.anchorPosition?.let { state.closestPageToPosition(it)?.prevKey?.plus(1) }
    }
}
```

**RemoteMediator（网络+数据库）：**
```kotlin
class ArticleRemoteMediator(
    private val api: ArticleApi,
    private val db: AppDatabase
) : RemoteMediator<Int, ArticleEntity>() {
    
    override suspend fun load(
        loadType: LoadType,  // REFRESH / PREPEND / APPEND
        state: PagingState<Int, ArticleEntity>
    ): MediatorResult {
        val page = when (loadType) {
            LoadType.REFRESH -> 1
            LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
            LoadType.APPEND -> getNextPage()
        }
        
        val response = api.getArticles(page)
        db.withTransaction {
            if (loadType == LoadType.REFRESH) db.articleDao().clearAll()
            db.articleDao().insertAll(response.articles)
        }
        return MediatorResult.Success(endOfPaginationReached = response.articles.isEmpty())
    }
}

// 配合使用
val pager = Pager(
    config = PagingConfig(pageSize = 20),
    remoteMediator = ArticleRemoteMediator(api, db)
) {
    db.articleDao().pagingSource()  // Room 自动生成 PagingSource
}
```

**PagingData 内部机制：**
- PagingData 是不可变的分页数据快照
- 内部通过 PageEvent（Insert/Drop/StateUpdate）流驱动 UI 更新
- PagingDataAdapter 的 DiffUtil 在后台线程计算差异

### 实战层

- **配置参数**：`PagingConfig(pageSize=20, prefetchDistance=5, initialLoadSize=40)`
- **加载状态**：`adapter.loadStateFlow` 监听 Loading/Error/NotLoading 状态，显示加载指示器
- **刷新**：`adapter.refresh()` 触发 REFRESH，重新从第一页加载
- **Compose 集成**：`collectAsLazyPagingItems()` + `LazyColumn`
- **坑点**：PagingSource 失效后需要返回新实例（Room 自动处理，自定义需要 `invalidate()`）

```kotlin
// Compose 中使用
@Composable
fun ArticleList(viewModel: ArticleViewModel) {
    val articles = viewModel.articles.collectAsLazyPagingItems()
    LazyColumn {
        items(articles.itemCount) { index ->
            articles[index]?.let { ArticleItem(it) }
        }
        // 加载状态
        when (articles.loadState.append) {
            is LoadState.Loading -> item { LoadingIndicator() }
            is LoadState.Error -> item { RetryButton() }
            else -> {}
        }
    }
}
```

### 延伸问题

- [[RecyclerView缓存机制]]
- [[Room编译时生成原理]]
- [[大数据量分页查询优化]]

## 记忆锚点

Paging3 三层：PagingSource 取数据 → Pager 转成 Flow → PagingDataAdapter 显示。RemoteMediator 是"网络拉到数据库，数据库喂给 UI"的中间人。
