<!--  -->
<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    
    <el-button @click="batchSave" v-if="draggable">批量保存</el-button>
     <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      @node-click="handleNodeClick"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            type="text"
            size="mini"
            @click="() => append(data)"
            v-if="node.level <= 2"
          >
            Append
          </el-button>
          <el-button
            type="text"
            size="mini"
            @click="() => remove(node, data)"
            v-if="node.childNodes.length == 0"
          >
            Delete
          </el-button>

          <el-button type="text" size="mini" @click="() => edit(data)">
            edit
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
import Vue from "vue";
//这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
  //import引入的组件需要注入到对象中才能使用
  components: {},
  data() {
    return {
      menus: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
      expandedKey: [],
      dialogVisible: false,
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: "",
        icon: "",
        productUnit: "",
      },
      dialogType: "", //区分修改或新增,
      title: "",
      maxLevel: 0,
      updateNodes: [], //所有修改的节点
      draggable: false,
      pCid:[]
    };
  },
  methods: {
    //批量删除
    batchDelete(){
        // let catIds=this.$refs.menuTree.getCheckedKeys();
        let catIds=[];
        let catName=[];
        let checkedNodes=this.$refs.menuTree.getCheckedNodes();
        for(let i=0;i<checkedNodes.length;i++){
          catIds.push(checkedNodes[i].catId)
          catName.push(checkedNodes[i].name)
        }
        this.$confirm(`是否批量删除【${catName}】菜单?`, "提示", {
          confirmButtonText: "确定",
          cancelButtonText: "取消",
          type: "warning",
        }).then(() => {
              this.$http({
                url: this.$http.adornUrl('/product/category/delete'),
                method: 'post',
                data: this.$http.adornData(catIds, false)
                }).then(({ data }) => { 
                    this.$message({
                      message:'菜单批量删成功',
                      type:'success'
                    })
                    this.getMenus();
                });
          }).catch(()=>{

          });

    },
    //批量保存
    batchSave() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单顺序修改成功！",
          type: "success",
        });
        //刷新菜单
        this.getMenus();
        //设置默认展开的菜单
        this.expandedKey = this.pCid;
        this.updateNodes = [];
        this.maxLevel = 0;
        // this.pCid=0;
      });
    },
    //拖拽成功调用的方法
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log(draggingNode, dropNode, dropType);
      //当前节点最新的父节点id
      let pCid = 0;
      let siblings = null;
      if (dropType == "inner") {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      } else if (dropType == "after" || dropType == "before") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      }
      this.pCid.push(pCid)

      //当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //如果遍历的是当前正在拖拽的节点
          let catLevel = draggingNode.level; //当前节点默认层级
          if (siblings[i].level != draggingNode.level) {
            //当前节点层级发生变化
            catLevel = siblings[i].level;
            //继续修改子节点的层级
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      console.log("updateNodes", this.updateNodes);
      
    },

    //改变子节点的层级并收集子节点的数据
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },

    allowDrop(draggingNode, dropNode, type) {
      //被拖动的当前节点以及所在的父节点总层数不能大于3
      //被拖动的当前节点总层数
      this.countNodeLevel(draggingNode);
      //当前正在拖动的节点+副节点所在的深度不大于3即可
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },

    countNodeLevel(node) {
      //找到所有子节点，求出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },

    //新增或修改方法
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },

    //提交修改操作
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      var data = { catId, name, icon, productUnit };
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(data, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功！",
          type: "success",
        });
        //关闭对话框
        this.dialogVisible = false;
        //刷新出新的菜单
        this.getMenus();
        //设置需要默认展开的菜单
        this.expandedKey = [this.category.parentCid];
      });
    },

    //修改分类
    edit(data) {
      this.dialogType = "edit";
      this.title = "修改分类";
      // console.log("要修改的数据",data)
      this.dialogVisible = true;
      //发送请求获取当前节点最新的请求
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        //请求成功
        // this.category=data.data
        //回显
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
    },

    //确认新添三级分类
    addCategory() {
      console.log("提交的三级分类数据", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单添加成功！",
          type: "success",
        });
        //关闭对话框
        this.dialogVisible = false;
        //刷新出新的菜单
        this.getMenus();
        //设置需要默认展开的菜单
        this.expandedKey = [this.category.parentCid];
      });
    },
    //弹出添加分类对话框
    append(data) {
      this.title = "添加分类";
      this.dialogType = "add";
      // console.log("append", data);
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1; //防止层级为字符串，可以先乘以1再加1
      this.category.name = "";
      this.category.catId = "";
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.sort = 0;
      this.category.showStatus = 1;
    },

    //删除分类
    remove(node, data) {
      console.log(node);
      console.log(data);
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            //   console.log("删除成功")
            this.$message({
              message: "菜单删除成功！",
              type: "success",
            });
            //刷新出新的菜单
            this.getMenus();
            //设置需要默认展开的菜单
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {
          this.$message({
            message: "已取消删除!",
            type: "warning",
          });
        });
    },
    //获取所有分类
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取到菜单数据", data.data);
        this.menus = data.data;
      });
    },
  },
  //监听属性 类似于data概念
  computed: {},
  //监控data中的数据变化
  watch: {},
  //生命周期 - 创建完成（可以访问当前this实例）
  created() {
    this.getMenus();
  },
  //生命周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {}, //如果页面有keep-alive缓存功能，这个函数会触发
};
</script>
<style scoped>
</style>