<template>
  <div>
    <el-switch
     v-model="draggable"
     active-text="开启拖拽"
     inactive-text="关闭拖拽">
    </el-switch>
    <el-button @click="batchSave" v-if="draggable">批量保存</el-button>
    <el-button type="danger" @click="batchDelete" >批量删除</el-button>
    <el-tree
    @node-drop="handleDrop"
    :data="menus"
    :props="defaultProps"
    :expand-on-click-node="false"
    @node-click="handleNodeClick"
    show-checkbox
    node-key="catId"
    :default-expanded-keys="expandedKey"
    :draggable="draggable"
    :allow-drop="allowDrop"
    >
    <span class="custom-tree-node" slot-scope="{ node, data }">
      <span>{{ node.label }}</span>
      <span>
        <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)" >Append</el-button>
        <el-button type="text" size="mini" @click="() => edit(data)">Edit</el-button>
        <el-button v-if="node.childNodes.length == 0" type="text" size="mini"  @click="() => remove(node, data)">Delete</el-button>
      </span>
    </span>
  </el-tree>
  <el-dialog title="title" :visible.sync="dialogVisible" width="30%" :close-on-click-modal="false">
      <el-form :model="category">                         
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addCategory">确 定</el-button>
      </span>
    </el-dialog>
 </div>
</template>

<script>
// 这里可以导入其他文件(比如:组件，工具 js，第三方插件 js，json文件，图片文件等等)
// 例如:import 《组件名称》 from '《组件路径》';

export default {
// import 引入的组件需要注入到对象中才能使用
  name: 'category',
  components: {},
  props: {},
  data () {
// 这里存放数据
    return {
      title: '',
      dialogType: '', // edit,add
      dialogVisible: false,
      menus: [],
      expandedKey: [],
      category: {name: '', parentCid: 0, catLevel: 0, showStatus: 1, sort: 0, icon: '', productUnit: '', catId: null},
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    }
  },
// 计算属性 类似于data概念
  computed: {},
// 监控 data 中的数据变化
  watch: {},
// 方法集合
  methods: {
    batchSave () {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({data}) => {
        this.$message({
          type: 'success',
          message: '菜单顺序修改成功'
        })
        this.getMenus()
        this.expandedKey = this.pCid 
        this.updateNodes = []
        this.maxLevel = 0
      })
    },
    batchDelete () {
      let checkNodes = this.$refs.tree.getCheckedNodes();
      let ids = [];
      let names = [];
      for (let i = 0; checkNodes.length; i++) {
        ids.push(checkNodes[i].cateId);
        names.push(checkNodes[i].name);
      }
      this.$confirm(`是否删除[${names}]菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      }).then( () => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: "post",
          data: this.$http.adornData(ids, false)
        }).then( () => {
          this.$message({
            message: "批量删除成功",
            type: "success"
          });
          this.getMenus();
        });
      }).catch(); 
    },
    handleDrop (draggingNode, dropNode, dropType, ev) {
      // 1.当前节点最新的父节点
      let pCid = 0
      let siblings = null
      if (dropType == 'before' || dropType == 'after') {
        pCid = dropNode.parent.data.catId == undefined ? 0 : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      } else {
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      }
      this.pCid.push(pCid)
      // 2.当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.cateLevel
          if (siblings[i].level != draggingNode.level) {
            catLevel = siblings[i].level 
            this.updateChildNodeLevel(siblings[i])
          }
          this.updateNodes.push({
            catId: siblings[i].data.cateId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel
          })
        } else {
          this.updateNodes.push({cateId: siblings[i].data.catId, sort: i})
        }
      }
    },
    updateChildNodeLevel (node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level
          })
          this.updateChildNodeLevel(node.childNodes[i])
        }
      }
    },
    allowDrop (draggingNode, dropNode, type) {
      var level = this.countNodeLevel(draggingNode)
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1
      if (type == 'inner') {
        return deep + dropNode.level <= 3
      } else {
        return deep + dropNode.parent.level <= 3
      }
    },
    countNodeLevel (node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level
          }
          this.countNodeLevel(node.childNodes)
        }
      }
    },
    handleNodeClick (data) {
      console.log(data)
    },
    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => {
        this.menus = data.data
      })
    },
    append (data) {
      this.dialogType = 'add'
      this.title = '添加分类'
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
      this.dialogVisible = true
      this.category.catId = null
      this.category.name = null
      this.category.icon = ''
      this.category.productUnit = ''
      this.category.sort = 0
      this.category.showStatus = 1
    },
    addCategory () {
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({data}) => {
        this.$message({
          type: 'success',
          message: '分类添加成功!'
        })
      })
      this.dialogVisible = false
      this.getMenus()
      this.expandedKey = [this.category.parentCid]
    },
    // 修改三级分类数据
    editCategory () {
      var {catId, name, icon, productUnit} = this.category
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId, name, icon, productUnit}, false)
      }).then(({data}) => {
        this.$message({
          type: 'success',
          message: '菜单修改成功!',
        })
        // 关闭对话框
        this.dialogVisible = false
        // 刷新出新菜单
        this.getMenus()
        // 设置需要默认展开的菜单
        this.expandedKey = [this.category.parentCid]
      }).catch(() => {})
    },
    edit (data) {
      this.dialogType = 'edit',
      this.title  = '修改分类',
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({data}) => {
        this.category.name = data.data.name
        this.category.catId = data.data.catId
        this.category.icon = data.data.icon
        this.category.productUnit = data.data.productUnit
        this.category.parentCid = data.data.parentCid
        this.dialogVisible = true
      })
    },
    submitData () {
      if (this.dialogType == 'add') {
        this.addCategory()
      }
      if (this.dialogType == 'edit') {
        this.editCategory()
      }
    },
    remove (node, data) {
      var ids = [data.catId]
      this.$confirm(`是否删除【${data.name}】当前菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'delete',
          data: this.$http.adornData(ids, false)
        }).then(({data}) => {
          this.$message({
            type: 'success',
            message: '菜单删除成功!'
          })
          // 刷新出新的菜单
          this.getMenus()
          this.expandedKey = [node.parent.data.catId]
        }).catch(() => {
          this.$message({
            type: 'info',
            message: '已取消删除'
          })
        })
      })
    }
  },
// 生命周期 - 创建完成(可以访问当前 this 实例)
  created () {
    this.getMenus()
  },
// 生命周期 - 挂载完成(可以访问 DOM 元素)
  mounted () {

  },
  beforeCreate () {}, // 生命周期 - 创建之前
  beforeMount () {}, // 生命周期 - 挂载之前
  beforeUpdate () {}, // 生命周期 - 更新之前
  updated () {}, // 生命周期 - 更新之后
  beforeDestroy () {}, // 生命周期 - 销毁之前
  destroyed () {}, // 生命周期 - 销毁完成
  activated () {} // 如果页面有 keep-alive 缓存功能，这个函数会触发
}
</script>
<style lang='scss' scoped>
// @import url(); 引入公共 css 类

</style>