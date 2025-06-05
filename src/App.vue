<template>
  <el-table
  :data="tableData"
  border
  style="width: 100%;margin-bottom: 20px;"
  row-key="kid"
  ref="table"
  :key="key"
  @expand-change="handleExpandChange"
  :tree-props="{children: 'children', hasChildren: 'hasChildren'}">
  <el-table-column
    prop="selfName"
    label="类别名称"
    >
  </el-table-column>
  <el-table-column
    prop="createDate"
    label="创建时间"
    >
  </el-table-column>
  <el-table-column
    width="55"
    draggable="true">
    <template #default="scope">
      <div class="drop">
        <img src="/drag.png"/>
      </div>
    </template>
  </el-table-column>
  <el-table-column
    width="100"
    draggable="true">
    <template #default="scope">
      {{ scope.row.id }}
    </template>
  </el-table-column>
</el-table>
</template>

<script>
import Sortable from 'sortablejs'
export default {
  data() {
    return {
      key: '',
      tempData: [],
      tableData: [],
      convertMap: {},
      convertMapChild: {},
      isDraggable: '',
      oldIndex: 0,
      newIndex: 0,
    }
  },
  methods: {
    handleExpandChange(row, expanded) {
      if (expanded && row.parentId === 0) {
        this.tableData.forEach(td => {
          if (td.id === row.id) {
            this.$refs.table.toggleRowExpansion(td, true)
          }else {
            this.$refs.table.toggleRowExpansion(td, false)
          }
        })
      }
    },
    //扁平化
    getTable(data) {
      data.forEach(d => {
        const obj = {
          ...d,
          cid: [],
        }
        delete obj.children
        this.tempData.push(obj)
        if (d.children && d.children.length > 0) {
          d.children.forEach(c => {
            obj.cid.push(c.id)
          })
          this.getTable(d.children)
        }
      })
    },
    //转为树
    // getTree() {
    //   const tempMap = {}
    //   const tempTree = []
    //   this.tempData.forEach(t => {
    //     const pid = t.parentId
    //     const id = t.id
    //     //父元素
    //     if (pid === 0) {
    //       delete t.cid
    //       //在tempMap中存在，说明有children
    //       if (tempMap[id]) {
    //         tempMap[id] = {
    //           ...tempMap[id],
    //           ...t,
    //         }
    //       }else {
    //         //在tempMap中存在，初始化对象
    //         tempMap[id] = t
    //         tempMap[id]['children']=[]
    //       }
    //       tempTree.push(tempMap[id])
    //     }else if (tempMap[pid]){ //子元素，在tempMap中存在
    //       tempMap[pid]['children'].push(t)
    //     }else { //子元素，在tempMap中不存在
    //       tempMap[pid] = {
    //         parentId: pid,
    //         children: [
    //           t
    //         ]
    //       }
    //     }
    //   })
    //   this.tableData = tempTree
    //   // this.key = new Date().getTime()
    //   console.log(tempTree);
    // },
    //table 父节点 id--index
    idIndexMap() {
      this.convertMap = this.tableData.reduce((acc, cur, index) => {
        return {
          ...acc,
          [cur.id] : index
        }
      }, {})
    },
    generateIdToIndexChild(children) {
      console.log(children);
      this.convertMapChild = children.reduce((acc, cur, index) => {
        return {
          ...acc,
          [cur.id] : index
        }
      }, {})
    },
    rowDrop() {
      const tb = document.querySelector('tbody')
      Sortable.create(tb, {
        handle: '.drop', 
        animation: 180,
        delay: 0,
        onEnd: ({ newIndex, oldIndex }) => {
          this.idIndexMap()
          this.newIndex = newIndex
          this.oldIndex = oldIndex
          console.log(newIndex, oldIndex);
          this.tempData = []
          this.getTable(this.tableData)
          const newData = this.tempData.at(newIndex)
          const delData = this.tempData.at(oldIndex)
          //第一层级移动
          if (delData.cid && delData.cid.length > 0 || delData.parentId === 0) {
            const delTdIndex = this.convertMap[delData.id]
            const newTdIndex = this.convertMap[newData.id]
            //整个table更新？？
            // const [dd] = this.tableData.splice(delTdIndex, 1)
            // this.key = new Date().getTime()
            // this.tableData.splice(newTdIndex, 0, dd)
            //优化
            const [dd] = this.tableData.splice(delTdIndex, 1)

            dd.kid = this.getUuid()

            dd.children.map(d => {
              d.kid = this.getUuid()
              return d
            })

            this.tableData.splice(newTdIndex, 0, dd)
            this.idIndexMap()
            this.categoryOrder()
          } else if (newData.parentId === delData.parentId) {
            //子层级移动
            const parentIndex = this.convertMap[newData.parentId]
            const parentData = this.tableData[parentIndex]
            const children = parentData.children
            this.generateIdToIndexChild(children)
            children.splice(this.convertMapChild[delData.id], 1)
            children.splice(this.convertMapChild[newData.id], 0, delData)
            this.generateIdToIndexChild(children)
            this.categoryOrderChild()
          }
        },
        onChoose: ({ oldIndex }) => {
          this.oldIndex = oldIndex
        },
        onMove: (customEvent, dragEvent) => {
          this.tempData = []
          this.getTable(this.tableData)
          this.newIndex = customEvent.related.rowIndex
          const newData = this.tempData.at(this.newIndex)
          const delData = this.tempData.at(this.oldIndex)
          if (newData.parentId !== delData.parentId) {
            return false
          }
        }
      })
    },
    categoryOrder() {
      const obj = Object.entries(this.convertMap).reduce((acc, cur) => {
        return [
          ...acc,
          {
            id: cur[0],
            order: cur[1]
          }
        ]
      }, [])
      console.log(obj);
    },
    categoryOrderChild() {
      const obj = Object.entries(this.convertMapChild).reduce((acc, cur) => {
        return [
          ...acc,
          {
            id: cur[0],
            order: cur[1]
          }
        ]
      }, [])
      console.log(obj);
    },
    getUuid () {
      return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function (c) {
        const r = Math.random() * 16 | 0
        const v = c === 'x' ? r : (r & 0x3 | 0x8)
        return v.toString(16)
      })
    },
  },
  mounted() {
    this.$nextTick(function () {
    // 仅在整个视图都被渲染之后才会运行的代码
      this.rowDrop()
    })
  },
  created() {
    // 仅在组件实例被创建之后才会运行的代码
    this.tableData = [
    {
        "id": 663,
        "kid": 663,
        "selfName": "1",
        "enabled": 0,
        "parentId": 0,
        "createDate": "2022-07-22 11:41:22",
        "children": [
            {
                "id": 664,
                "kid": 664,
                "selfName": "1.1",
                "enabled": 0,
                "parentId": 663,
                "createDate": "2022-07-22 11:42:18",
                "children": []
            },
            {
                "id": 665,
                "kid": 665,
                "selfName": "1.2",
                "enabled": 0,
                "parentId": 663,
                "createDate": "2022-07-22 11:44:06",
                "children": []
            },
            {
                "id": 666,
                "kid": 666,
                "selfName": "1.3",
                "enabled": 0,
                "parentId": 663,
                "createDate": "2022-07-22 12:03:54",
                "children": []
            }
        ]
    },
    {
        "id": 1,
        "kid": 1,
        "selfName": "2",
        "enabled": 0,
        "parentId": 0,
        "createDate": "2022-07-22 11:41:22",
        "children": [
            {
                "id": 2,
                "kid": 2,
                "selfName": "2.1",
                "enabled": 0,
                "parentId": 1,
                "createDate": "2022-07-22 11:42:18",
                "children": []
            },
            {
                "id": 3,
                "kid": 3,
                "selfName": "2.2",
                "enabled": 0,
                "parentId": 1,
                "createDate": "2022-07-22 11:44:06",
                "children": []
            },
            {
                "id": 4,
                "kid": 4,
                "selfName": "2.3",
                "enabled": 0,
                "parentId": 1,
                "createDate": "2022-07-22 12:03:54",
                "children": []
            }
        ]
    }
  ]
  }
}
</script>
