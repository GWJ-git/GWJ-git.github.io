---
title: 'el-admin框架学习总结(1)'
date: 2022-09-22T11:43:55+08:00
draft: false
toc: true
images: null
categories:
  - 框架学习
tags:
  - el-admin
  - vue
  - '学习笔记'
slug: ''
---

个人学习思路，先模仿给出的那些管理页面写，大致清楚如何写一个页面后，根据页面加载或生命周期看每一步在做什么
个人感觉 crud.js(src/components/Crud/crud.js)值得一看,是该后台系统 各种增删改查的核心
其他动态路由等也是学习点，暂时未总结

### 项目文件夹结构

```
-node_modules		依赖安装文件夹（要进行git忽略）
-dist				build后的文件夹（要进行git忽略）
-public 			主要放置入口index.html文件和项目LOGO
-src
--api 				**请求api封装（封装请求-常用）
--assets			静态资源
--components		**自定义组件封装（封装所需要的组件-常用）
--router			**路由（路由跳转-常用）
--store				vuex配置
--utils				通用工具封装
--views				**页面开发（主要的页面开发-常用）
-App.vue			vue根页面
-main.js			vue全局配置
-settings.js		项目配置
-.env.development	开发环境配置
-.env.production	生产环境配置
-package.json		依赖配置
-vue.config.js      vue项目环境配置文件
```


页面流程



~~~js
import { mapGetters } from 'vuex'
import CRUD, { presenter, header, form, crud } from '@crud/crud'
// 默认表单，设置数据格式  最好与接口返回的格式一致，直接暴露，不要封装
const defaultForm = {
  id: null,
  username: null,
  // appId: null   自定义id字段，要在cruds 中添加idField
}
export default {
  name: 'User',
  components: {
    Treeselect,
  },
  // 自定义的 property 
  cruds() {
    // crud.js中的CRUD方法
    return CRUD({
      title: '用户',
      url: 'api/users',
      // idField: 'appId'  自定义id字段
      crudMethod: { ...crudUser }
    })
  },
  // 混入
  mixins: [presenter(), header(), form(defaultForm), crud()],
  data() {
    // 自定义验证
    const validPhone = (rule, value, callback) => {
      if (!value) {
        callback(new Error('请输入电话号码'))
      } else if (!isvalidPhone(value)) {
        callback(new Error('请输入正确的11位手机号码'))
      } else {
        callback()
      }
    }
    return {
      deptName: '',
      //权限
      permission: {
        add: ['admin', 'user:add'],
      },
      // 验证规则  
      rules: {
        username: [
          { required: true, message: '请输入用户名', trigger: 'blur' },
          {
            min: 2,
            max: 20,
            message: '长度在 2 到 20 个字符',
            trigger: 'blur'
          }
        ]
  },
  computed: {
    //返回store中的user对应的字段
    ...mapGetters(['user'])
  },
  created() {
    this.crud.msg.add = '新增成功，默认密码：123456'
  },
  mounted: function() {
    const that = this
    window.onresize = function temp() {
      that.height = document.documentElement.clientHeight - 180 + 'px;'
    }
  },
  methods: {
    //crud钩子在callVmHook中调用
    [CRUD.HOOK.afterAddError](crud) {
      this.afterErrorMethod(crud)
    },
    // 新增与编辑前做的操作
    [CRUD.HOOK.afterToCU](crud, form) {
        //.....
    },
    // 打开编辑弹窗前做的操作
    [CRUD.HOOK.beforeToEdit](crud, form) {
        //........
    },
}
~~~

15行，自定义option

~~~js
  // 自定义的 property 
  cruds() {
    // crud.js中的CRUD方法
    return CRUD({
      title: '用户',
      url: 'api/users',
      // idField: 'appId'  自定义id字段
      crudMethod: { ...crudUser }
    })
  },
~~~

`mixins: [presenter(), header(), form(defaultForm), crud()]` 

混入模式下

`presenter()`

~~~js
function presenter(crud) {
  if (crud) {
    console.warn('[CRUD warn]: ' + 'please use $options.cruds() { return CRUD(...) or [CRUD(...), ...] }')
  }
  return {
    data() {
      // 在data中返回crud，是为了将crud与当前实例关联，组件观测crud相关属性变化
      return {
        crud: this.crud
      }
    },
    beforeCreate() {
      // 储存主页实例
      this.$crud = this.$crud || {}
      //此处将实例里cruds中返回的CRUD函数初始化
      let cruds = this.$options.cruds instanceof Function ? this.$options.cruds() : crud
      if (!(cruds instanceof Array)) {
        cruds = [cruds]
      }
      cruds.forEach(ele => {
        if (this.$crud[ele.tag]) {
          console.error('[CRUD error]: ' + 'crud with tag [' + ele.tag + ' is already exist')
        }
        this.$crud[ele.tag] = ele
        ele.registerVM('presenter', this, 0)
      })
      this.crud = this.$crud['defalut'] || cruds[0]
    },
    methods: {
      parseTime
    },
    created() {
      for (const k in this.$crud) {
        if (this.$crud[k].queryOnPresenterCreated) {
          // 默认查询
          this.$crud[k].toQuery()
        }
      }
    },
    destroyed() {
      for (const k in this.$crud) {
        this.$crud[k].unregisterVM('presenter', this)
      }
    },
    mounted() {
      // 如果table未实例化（例如使用了v-if），请稍后在适当时机crud.attchTable刷新table信息
      if (this.$refs.table !== undefined) {
        this.crud.attchTable()
      }
    }
  }
}
~~~

`header(),pagination(),form(),crud`


关于页面中`[CRUD.HOOK.afterToCU](crud, form) `之类的钩子如何执行
之前一直不太清楚是如何触发的，今天又翻了翻crud.js
本以为不在`callVmHook`中触发的，没想到还是在这里
js基础还是要经常学习巩固
~~~js
//[CRUD.HOOK.beforeToEdit]如何触发
[CRUD.HOOK.afterToCU](crud, form) {
        //.....
}
-------------
//crud.js
// hook VM
function callVmHook(crud, hook) {
  if (crud.debug) {
    console.log('callVmHook: ' + hook)
  }
  const tagHook = crud.tag ? hook + '$' + crud.tag : null
  let ret = true
  const nargs = [crud]
  for (let i = 2; i < arguments.length; ++i) {
    nargs.push(arguments[i])
  }
  // 有些组件扮演了多个角色，调用钩子时，需要去重
  const vmSet = new Set()
  crud.vms.forEach(vm => vm && vmSet.add(vm.vm))
  vmSet.forEach(vm => {
    if (vm[hook]) {
      ret = vm[hook].apply(vm, nargs) !== false && ret
    }
    if (tagHook && vm[tagHook]) {
      ret = vm[tagHook].apply(vm, nargs) !== false && ret
    }
  })
  return ret
}
-----------------
//关键在于apply方法  方法重用
//vm[hook].apply(vm, nargs)
//将vm[hook]也就是实例中的[CRUD.HOOK.beforeToEdit]之类的方法在vm和nargs(也就是[crud])中重用
~~~

