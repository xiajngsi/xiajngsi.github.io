---
title: schema 动态表单设计及部分实现
---
# 背景

为什么需要动态表单及动态交互怎么用 schema 设计？先看以下场景，比如表单中有 A、B 两个字段

### 1. 简单交互

A 字段的值控制 B 字段的显隐、属性、值。
比如：学校对应班级，幼儿园只有大中小班，小学有 1-6 年级。
在表单中，选择小学后，对应出现 1-6 年级，而不是 大中小班

国家 字段 A，收费 B 字段，单位作为 B 字段组件的属性
A 改变后，B 字段的属性需要跟着变

以城市省名 A、城市名 B选择举例。
由于每个省对应的市列表都不一样，选择省后，需要清空之前选过的城市。

### 2. 下拉选择框

由 A 触发 B 列表刷新。
以上面省市区的列子来说。
由于每个省对应的市列表都不一样，选择省后，需要动态获取市列表。

## 其他功能

校验：必填、正则等
编辑态的处理

## schema 设计

```javascript
{
  name: string, // 字段名
  label: string, // 表单 label
  type: string, // 当前字段所使用的组件
  show: 'A === "testA" || B === "testB"', // 当前字段展示的表达式
  required: boolean, // 是否必填
  readonly: boolean, // 是否只读
  initialValue: any, // 初始值
  tooltip: string | {title: string, icon: ReactNode}, //字段提示
  listeners: [
    {
      watch: [ "type"],
      context: { defaultScriptType },
      set: {
        mode: "type || defaultScriptType"
      }
    },
    {
      watch: [ "type", "params"],
      context: { defaultScriptType, getHeaderValue },
      set: {
        headerValue: `getHeaderValue(defaultScriptType, type, params)`
      }
    }
  ], // 字段之间的联动
  customPropsDefaultValue: {[string]: any}, //设置属性的初始值
  customProps: { // 每个渲染组件有不同的属性
    optionsTransfer: (options) => {return options.filter}, // select 控件属性，过滤或者字段处理, 一般在 custom-component-map 里定义
    options: [] // restApi string 或者数组或者 object
    service: Function, // 获取下拉选项的方法
    afterOptions: (options: any) => void // 获取数据后的处理
  },
  path: '/api/{provinceId}/city', // 接口请求地址
  params: 'provinceId={provinceId}', // 接口请求参数
  validations: rule[] // async-validator 可定义的规则 https://github.com/yiminghe/async-validator
}
```

### 简单交互之 A 字段的值控制 B 字段的显隐

通过 show 属性， 值是字符串表达式，里面可以直接写当前表单的所有变量和传入的全局上下文，下面是实现。

```javascript
(formValues) => {
    try {
      return new Function('values', `with(values){return ${show}}`)({...varPreProcess(fields), ...formValues, ...globalContext})
    } catch (error) {
      console.error(error)
    }
  }
```

这种形式对比 apply 更友好，表达式更简单，但是需要对内部变量做预处理，varPreProcess 就是对变量的预处理，下面是 varPreProcess 的实现。

```javascript
// fields 是所有的字段
export const varPreProcess = (fields) => {
  const varPreset = {}
  fields.forEach((field) => {
     const { name } = field;
     varPreset[name] = undefined
  });
  return varPreset
}
```

### A 字段值控制 B 字段的属性或者值

这两个一起讲是因为字段设置大部分类似，稍有差别。

```javascript
 listeners: [
    {
      watch: [ "type"], // 当前字段需要监听的字段
      context: { defaultScriptType }, // 需要的上下文
      set: {
        mode: "type || defaultScriptType" // 当前属性
      }
    },
   {
      watch: [ "type"], // 当前字段需要监听的字段
      context: { defaultScriptType }, // 需要的上下文
      set: {
        value: "type || defaultScriptType" // 当前值的表达式
      }
    },
    {
      watch: [ "type", "params"],
      context: { defaultScriptType, getHeaderValue },
      set: {
        headerValue: `getHeaderValue(defaultScriptType, type, params)`
      }
    }
  ],
```

set 的属性会订阅事件，watch 字段变化时会发布事件，从而达到更新。这里有个点注意的是：listener 只是字段变化的时候会处理，对于初始值 value 是通过 initValue 设置，属性的初始值需要自行处理

### 下拉选择的处理

通过传入 path、params 或者 customProps 里面的 service 方法来获取下拉选项或者通过 customProps 可以直接传入 options 选项，path 和 params 可以用 “{}” 包裹变量，这些变量是表单内的字段，当变量变化时会重新发起请求。
optionsTransfer 可以通过外部修改该选项，如对格式的转换，过滤等等。
customProps 中还有个参数 afterOptions， 这个参数是为了处理获取请求之后想要拿到 options

第一步是分析 path 和 params 的变量，并转换成 listeners 做监听

```javascript
const getPathListeners = ({config = {}, name, customProps, context}) => {
  const str = config[name]
  const regex = /(?<={).*?(?=})/g
  if(str && /(?<={).*?(?=})/.test(str)) {
    let watchs = []
    // 提取 path 和 params 里面的变量
    let execResult = regex.exec(str)
    while (execResult !== null) {
      watchs.push(execResult[0])
      str = str.slice(0, execResult.index - 1 ) + '$' + str.slice(execResult.index - 1)
      execResult = regex.exec(str)
    }
    const setStr = "`" + str + "`"
    if(watchs.length) {
      config.customPropsDefaultValue = config.customPropsDefaultValue || {}
      const valueIsNotExist = watchs.find((watchKey) => !context[watchKey]) 
      if(!valueIsNotExist) {
        config.customPropsDefaultValue[name] = new Function('values', `with(values){return ${setStr}}`)(context);
      }
      // 生成 listeners
      return {
        watch: watchs,
        set: {
          [name]: setStr,
   value: undefined
        }
      }
    }
    
  } else if (str) {
    customProps[name] = str
  }
}

```

第二部是在组件内比如 select 组件、cascader 等组件内部发起请求，下面是我们项目内提取的 hooks，可直接享用。

```javascript
export const serviceOptionHooks = ({optionsTransfer, path, param, options: propsOptions = [], afterOptions, service}: IServiceOptionParam) => {
  const [options, setOptions] = React.useState([]);
  const [fetching, setFetching] = React.useState(false);
  const firstRequest = useRef(true)
  useEffect(() => {
    (async () => {
      let result: any[] = propsOptions
      if(path || service){
        setFetching(true);
        try {
          if(service) {
            const res = await service()
            result = res.options
          } else if (path) {
            const res = await customAgent.get(path).query(param)
            result = res.body
          }
          if (firstRequest.current && afterOptions) {
            firstRequest.current = false
            afterOptions(result)
          }
        } catch (error) {
          console.error('error', error)
        } finally {
          setFetching(false); // always close the resource
       }
      }
      if(optionsTransfer) {
        setOptions(optionsTransfer(result as any[]))
      } else { 
        setOptions(result)
      }
    })()
  }, [path, param, propsOptions.length && propsOptions, afterOptions]);
  return {options, fetching}
}
```

### 编辑态的处理

编辑态和新增态的区别是初始值的处理，值的初始化处理 initValue，属性初始化的处理 customPropsDefaultValue，对于 B 字段依赖 A 字段下拉选择回来的数据处理，比如 options 内的其他参数，则可以用 afterOptions 处理

## 总结

本文只是阐述动态表单一部分功能的设计上和实现，可针对各种不同基础组件，内容是表单的设计还包括特殊类型入下来选择组件的设计，具体的实现可根据不同基础框架自行封装。
需要有实现直接用的可以参考 <https://x-render.gitee.io/form-render>（基于 antd），开箱即用。
