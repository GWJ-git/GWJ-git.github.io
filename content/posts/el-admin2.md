---
title: 'el-admin框架学习总结(2)'
date: 2022-09-30T08:53:55+08:00
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

混入基础 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210403085118483.png#pic_center)


`mixins: [presenter(), header(), form(defaultForm), crud()]` 

在这些混入完毕后，`presenter()`中的`created()`钩子触发`toQuery()`

接着触发`refresh()`方法

```js
// 刷新
refresh() {
    if (!callVmHook(crud, CRUD.HOOK.beforeRefresh)) {
        return;
    }
    return new Promise((resolve, reject) => {
        //crud加载状态，表格中的加载
        crud.loading = true;
        // 请求数据
        //crud.url在cruds()中初始化
        initData(crud.url, crud.getQueryParams()).then(res => {
            // getTable方法 return this.findVM('presenter').$refs.table;
            const table = crud.getTable();
            if (table && table.lazy) { // 懒加载子节点数据，清掉已加载的数据
                table.store.states.treeData = {};
                table.store.states.lazyTreeNodeMap = {};
            }
            //表格绑定数据  期望数据 res.data中的数据数组或res.data.records（分页）
            crud.page.total = res.data.pages === undefined ? 0 : res.data.total;
            crud.data = res.data.pages === undefined ? res.data : res.data.records;
            // 重置数据状态  编辑和删除的状态值
            crud.resetDataStatus();
            // time 毫秒后显示表格
            setTimeout(() => {
                crud.loading = false;
                callVmHook(crud, CRUD.HOOK.afterRefresh);
            }, crud.time);
            // 清除上次搜索的条件
            crud.query.startTime = null;
            crud.query.endTime = null;
            resolve(data);
        }).catch(err => {
            crud.loading = false;
            reject(err);
        });
    });
},
```

`toquery()`中有`refresh()`方法

主要刷新表格及清空`DateRangePicker`组件的查询条件

对于查询问题

~~~js
/**
     * 获取查询参数
     */
getQueryParams: function() {
    // 清除参数无值的情况
    Object.keys(crud.query).length !== 0 && Object.keys(crud.query).forEach(item => {
        if (crud.query[item] === null || crud.query[item] === '') crud.query[item] = undefined;
        // 对creatime参数进行修改，createtime数组改为 startTime与endTime
        if (crud.query[item] !== undefined && item === 'createTime') {
            const timeArr = crud.query[item];
            crud.query['startTime'] = timeArr[0];
            crud.query['endTime'] = timeArr[1];
        }
    });
    Object.keys(crud.params).length !== 0 && Object.keys(crud.params).forEach(item => {
        if (crud.params[item] === null || crud.params[item] === '') crud.params[item] = undefined;
    });
    return {
        current: crud.page.page,
        size: crud.page.size,
        orders: JSON.stringify(crud.sort),
        ...crud.query,
        ...crud.params
    };
},
~~~

有时我们对于时间范围的需求需要不同的参数值，而`DateRangePicker`组件绑定的`createTime`

的只能转化为`startTime`和`endTime`,改组件会影响原有功能，重新写一个又不至于

所以我们可以在`getQueryParams`方法中添加同级判断（避免影响原有功能），来处理我们自己的需求

~~~js
 // 对creatime参数进行修改，createtime数组改为 startTime与endTime
if (crud.query[item] !== undefined && item === 'createTime') {
    const timeArr = crud.query[item];
    crud.query['startTime'] = timeArr[0];
    crud.query['endTime'] = timeArr[1];
}
// 以`DateRangePicker`组件绑定query.Time为例
if (crud.query[item] !== undefined && item === 'Time') {
    // 按自己的需求更改 下面字段名
    const timeArr = crud.query[item];
    crud.query['beginTime'] = timeArr[0];
    crud.query['endTime'] = timeArr[1];
    -----
    //或直接传递数组,按需处理
    crud.query['times'] = crud.query[item]
}
~~~



然后是新增功能

通常用的是`crud.operation`组件

新增按钮触发

~~~js
/**
     * 启动添加
     */
toAdd() {
    // 重置表单
    crud.resetForm();
    //验证crud钩子
    if (!(callVmHook(crud, CRUD.HOOK.beforeToAdd, crud.form) && callVmHook(crud, CRUD.HOOK.beforeToCU, crud.form))) {
        return;
    }
    // 修改新增状态为PREPARED
    crud.status.add = CRUD.STATUS.PREPARED;
    // 调用生命周期钩子
    callVmHook(crud, CRUD.HOOK.afterToAdd, crud.form);
    callVmHook(crud, CRUD.HOOK.afterToCU, crud.form);
},
~~~

~~~js
status: {
    add: CRUD.STATUS.NORMAL,
	edit: CRUD.STATUS.NORMAL,
	// 添加或编辑状态
    //cu-get方法返回新增或编辑的状态值
	get cu() {
		if (this.add === CRUD.STATUS.NORMAL && this.edit === CRUD.STATUS.NORMAL) {
			return CRUD.STATUS.NORMAL;
		} else if (this.add === CRUD.STATUS.PREPARED || this.edit === CRUD.STATUS.PREPARED) {
			return CRUD.STATUS.PREPARED;
		} else if (this.add === CRUD.STATUS.PROCESSING || this.edit === CRUD.STATUS.PROCESSING) {
			return CRUD.STATUS.PROCESSING;
		}
		throw new Error('wrong crud\'s cu status');
	},
    // 标题，弹窗的标题
    get title() {
    	return this.add > CRUD.STATUS.NORMAL ? `新增${crud.title}` : this.edit > CRUD.STATUS.NORMAL ? `编辑${crud.title}` : crud.title;
    }
},
~~~

`:visible.sync="crud.status.cu > 0"`弹窗组件绑定的cu，此时显示

提交表单执行`crud.submitCU`方法--进行表单验证，`.$refs['form']`,并根据状态执行` crud.doAdd()`方法（新增按钮toadd修改新增状态值）

~~~js
/**
     * 执行添加
     */
doAdd() {
    if (!callVmHook(crud, CRUD.HOOK.beforeSubmit)) {
        return;
    }
    //修改新增状态
    crud.status.add = CRUD.STATUS.PROCESSING;
    //请求接口
    crud.crudMethod.add(crud.form).then(() => {
        //修改状态为正常
        crud.status.add = CRUD.STATUS.NORMAL;
        //重置表单
        crud.resetForm();
        //成功通知
        crud.addSuccessNotify();
        callVmHook(crud, CRUD.HOOK.afterSubmit);
        //查询--刷新--刷新表格
        crud.toQuery();
    }).catch(() => {
        crud.status.add = CRUD.STATUS.PREPARED;
        callVmHook(crud, CRUD.HOOK.afterAddError);
    });
},
~~~

编辑是一样的流程

