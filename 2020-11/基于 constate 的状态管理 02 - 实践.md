# 基于 constate 的状态管理 - 实践



## 通用场景

应对如 CURD 这类的通用场景，基本思路应该是配置对象多余逻辑代码的，再通过预留自定义的入口，满足定制。

因此，对于配置化结构要如何设计就显得特别重要，跟某个具体产品的结合也会导致细微的差异。同时，提供可配置的范围边界可能也是不固定的。只有通过具体某个项目的实践经验来设计，并且在开发过程中不断优化或重构。



## 列表

列表的数据抽象相对清晰，如分页加载、条件搜索，最多会加上排序数据，像是多选后的操作、导入导出等，不在数据层管理，一是场景不够通用，二是这些功能一般会和具体 UI 组件耦合，干脆丢给视图层去自己实现。

这里为了方便理解，我们只实现分页加载和条件搜索。

首先，定义我们的配置结构：

```tsx
export interface UseListOptions<T, S> {
  url: string | ((search: S) => string | null)
  getDefaultSearch?: () => Partial<S>						//搜索表单的默认值
  parseFetchedData?: (data, search: S) => { items: T[]; total: number }
  size?: number
}
```

然后确定自定义 hook 需要是对外暴露的数据和方法：

```tsx
export interface ListContext<T = any, S = any> {
  items: T[]
  index: { value: number }
  size: number
  total: number
  loading: boolean
  search: Record<string, any>

  fetch: (options?: { reset: boolean }) => Promise<any>
  onIndexChange: (current: number) => void
  onSearchSubmit: (search: Partial<S>) => void
  onSearchReset: () => S
}
```

接下来就是我们的工厂方法 createList，通过配置信息 `UseListOptions` 实现逻辑定制，最后用 constate 将 hook 包装导出：

```ts
export function createList<T = any, S = any>(options: Readonly<UseListOptions<T, S>>) {
  function hook(): ListContext<T, S> {
    const [index, setIndex] = useState({ value: 1 })
    const [size] = useState(options.size || 30)
    const [total, setTotal] = useState(0)
    const [items, setItems] = useState<T[]>([])
    const [loading, setLoading] = useState(false)
    const [search, setSearch] = useState<S>(
      (options.getDefaultSearch ? options.getDefaultSearch() : {}) as S
    )
    
    async function fetch() {
      const url = typeof options.url === 'function' ? options.url(search) : options.url
      setLoading(true)
      try {
        const { data } = await GET(url, {
          data: {
            pageIndex: Math.max(index.value, 1),
            pageSize: size,
            ...search
          }
        })
        const { items, total } = parseFetchedData(data)
        setItems(items)
        setTotal(total)
      } catch (e) {
        console.error(e)
      } finally {
        setLoading(false)
      }
    }

    function parseFetchedData(data) {
      if (options.parseFetchedData) {
        return options.parseFetchedData(data, search)
      }
      const items = data.items || data
      const total = data.totalCount || items.length
      return { items, total }
    }

    // useUpdateEffect 是来自 ahooks 的 API，初始化过程不会触发 effect
    useUpdateEffect(() => {
			fetch()
    }, [index])

    useUpdateEffect(() => {
      // index 使用包装对象，强制触发 fetch
      setIndex({ value: 1 })
    }, [search])

    return {
      items,
      index,
      size,
      total,
      loading,
      search,
      fetch,
      onIndexChange: current => setIndex({ value: current }),
      onSearchSubmit: search => setSearch(prevState => ({ ...prevState, ...search })),
      onSearchReset: () => {
        const val = (options.getDefaultSearch
          ? options.getDefaultSearch()
          : {}) as S
        setSearch(val)
        return val
      }
    }
  }

  const [ListProvider, useListContext] = constate(hook)
  ListProvider.displayName = 'ListProvider'
  return { ListProvider, useListContext }
}
```



## 编辑

理解了 createList，实现 createEdit 就没啥难度了。直接看配置定义:

```ts
// <T>: FormDataType
// <P>: ParamsType 编辑入参，由视图层调用时传入，一般是列表中的某一条记录

type ValueOrValueFunction<P, V> = V | ((params: P) => V)

export interface UseEditOptions<T, P> {
  // *新增* 获取表单的默认值
  getDefaultFormData?: () => any
  // *编辑* 获取资源
  fetch?: {
    url: ValueOrValueFunction<P, string>
    data?: ValueOrValueFunction<P, any>
    // 可选的提供预处理方法，将获取资源转换成 FormData
    parser?: (data: any, params: P) => T
  }
  // *编辑* 提交数据
  submit?: {
    url: ValueOrValueFunction<{ isEdit: boolean; data: T; params: P }, string>
    // 可选的数据处理方法，将表单数据转换回原始资源类型
    parser?: (data: T, params: P) => any
  }
  // *删除*
  remove?: {
    url: ValueOrValueFunction<P, string>
  }
}
```

然后是对外暴露的数据和方法：

```ts
export interface EditContext<T, P> {
  visible: boolean        // 控制编辑对话框
  isEdit: boolean         // 标示表单处于新增或编辑状态
  data: T                 // 表单数据
  params: P               // 编辑入参
  loading: boolean
  saving: boolean

  onAdd: (params?: P) => void    // 新增数据
  onEdit: (params: P) => void    // 编辑数据，P 一般是列表中的某一条记录
  onSubmit: (data: T) => void    // 表单提交，data 一般由表单组件负责维护
  onCancel: () => void           // 取消编辑
  onRemove: (data) => void       // 删除数据

  connect: (list: ListContext) => void  // 建立到 list 的关联，刷新列表脏数据
}
```

最后，实现下 createEdit 工厂：

```ts
export function createEdit<FormDataType = any, ParamsType = FormDataType>(
  options: Readonly<UseEditOptions<FormDataType, ParamsType>>
) {
  function hook(): EditContext<FormDataType, ParamsType> {
    const [visible, setVisible] = useState(false)
    const [isEdit, setIsEdit] = useState(false)
    const [data, setData] = useState<FormDataType>({} as any)
    const [params, setParams] = useState<ParamsType>({} as any)
    const [loading, setLoading] = useState(false)
    const [saving, setSaving] = useState(false)

    function onAdd(params?) {
      setParams(params)
      setData(options.getDefaultFormData ? options.getDefaultFormData(params) : {})
      setIsEdit(false)
      setVisible(true)
    }

    async function onEdit(params: ParamsType) {
      setParams(params)
      setIsEdit(true)
      if (options.fetch) {
        setVisible(true)
        setLoading(true)
        const fetchUrl = getFunctionalValue(options.fetch.url, params)
        const fetchParams = options.fetch.data && { data: options.fetch.data(params) }
        
        // 省略 GET 请求
        
        setData(options.fetch.parser ? options.fetch.parser(responseData, params) : responseData)
        setLoading(false)
      } else {
        setData((params as any) as FormDataType)
        setVisible(true)
      }
    }

    async function onSubmit(formData) {
      if (!options.submit) {
        console.error('[useEdit] options.submit undefined')
        return
      }
      const _data = { ...data, ...formData }
      setSaving(true)
      
      const submitUrl = getFunctionalValue(options.submit.url, { isEdit, data: _data, params })
			const submitData = options.submit.parser ? 
            options.submit.parser(_data, params) : _data
      
      // 省略 POST 或 PUT
      
      setVisible(false)
      setSaving(false)
      message('保存成功')
      requestListReload({ reset: true })
    }

    function onCancel() {...}

    async function onRemove(data) {...}

    function getFunctionalValue(val, params) {
      return typeof val === 'function' ? val(params) : val
    }

    // inject listContext
    const listRef = useRef<ListContext>()

    function connect(list: ListContext) {
      listRef.current = list
    }

    function requestListReload(options?: { reset: boolean }) {
      listRef.current && listRef.current.fetch(options)
    }

    return {...}
  }

  const [EditProvider, useEditContext] = constate(hook)
  EditProvider.displayName = 'EditProvider'
  return { EditProvider, useEditContext }
}
```



## 使用示例

```tsx
const { ListProvider, useListContext } = createList({
  url: () => '/queryAll'
})

const { EditProvider, useEditContext } = createEdit({
  fetch: {
    url: '/getOne'
  },
  submit: {
    url: '/getOne'
  },
  remove: {
    url: '/getOne'
  }
})

const Page = () => {
  const list = useListContext()
  const edit = useEditContext()
  edit.connect(list)

  useMount(() => { list.fetch() })
  
  return (
    <Button onClick={() => edit.onAdd()}>
      新增
    </Button>
		<SearchTable list={list} onRemoveConfirm={edit.onRemove} />
    <Edit />
  )
}

const Edit = () => {
  const edit = useEditContext()
  return (
  	<EditModal edit={edit} onSubmit={edit.onSubmit}>
      ...
    </EditModal>
  )
}

export default () => (
  <ListProvider>
    <EditProvider>
      <Page />
    </EditProvider>
  </ListProvider>
)
```

可以看到，示例中的 `SearchTable`、`EditModel` 都直接依赖了 ListContext 和 EditContext，由他们自己内部决定怎么去消费和操作 Context。实现比较简单，这里就不再展开了，可前往下方的项目模版中查看。



## 最后

小小总结一下，这套通用业务的封装，在到目前为止服务的正式和边缘项目有7、8个，总体还是足够满足各种需求的，但是每个项目又多多少少都要做些定制后者修改，从 contstate 工厂到与之相匹配的特化组件都是如此。

就像开头所说的，每个项目都伴随很大的不确定性，靠个人经验很难完全预测，这也打消了我天真地想把这套实践包装成框架的想法，可能这篇文章才是最好产物。



下一节，我们将结合一个相对复杂的业务场景，尝试数据分层的管理方式。

[基于 constate 的状态管理 - 复杂场景]() 👈




## 相关项目

- [constate-crud-example](https://github.com/gouflv/constate-crud-example) 一个 constate 结合 and design 的项目模版

- [yinz](https://github.com/gouflv/yingz) 真实案例



