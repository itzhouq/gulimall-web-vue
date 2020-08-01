<!--  -->
<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      draggable
      :allow-drop="allowDrop"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(node, data)"
          >Append</el-button>
          <el-button type="text" size="mini" @click="edit(data)">Edit</el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >Delete</el-button>
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
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>

      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData()">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  components: {},
  data() {
    return {
      menus: [],
      title: "",
      maxLevel: 0,
      dialogType: "", // add edit
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 1,
        productUnit: "",
        icon: "",
        catId: null,
      },
      dialogVisible: false,
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  computed: {},
  watch: {},
  methods: {
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取到菜单数据", data.data);
        this.menus = data.data;
      });
    },
    allowDrop(draggingNode, dropNode, type) {
      // 1. 被拖动的当前节点以及所在父节点总层数不能大于3
      // 1）被拖动的当前节点的总层数
      console.log("allowDrop: ", draggingNode, dropNode, type);
      this.countNodeLevel(draggingNode.data);
      // 当前正在拖动的节点 + 父节点所在的深度不大于3即可
      let deep = (this.maxLevel - draggingNode.data.catLevel) + 1;
      console.log("深度：", deep);

      if(type == "inner") {
        return (deep + dropNode.level) <= 3;
      } else {
        return (deep + dropNode.parent.level) <= 3;
      }
    },
    countNodeLevel(node) {
      // 找到所有子节点，求出最大深度
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          this.countNodeLevel(node.children[i]);
        }
      }
    },
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    // 修改三级分类数据
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功",
          type: "success",
        });
        // 关闭对话框
        this.dialogVisible = false;
        // 刷新出新的菜单
        this.getMenus();
        // 设置需要默认展开的菜单
        this.expandedKey = [this.category.parentCid];
      });
    },
    edit(data) {
      console.log("要修改的数据： ", data);
      this.dialogVisible = true;
      this.dialogType = "edit";
      this.title = "修改分类";
      // 发送请求获取当前节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        // 请求成功
        console.log("要回显的数据", data);
        this.category.name = data.category.name;
        this.category.catId = data.category.catId;
        this.category.icon = data.category.icon;
        this.category.productUnit = data.category.productUnit;
        this.category.parentCid = data.category.parentCid;
      });
    },
    // 添加的三级分类
    addCategory() {
      console.log("添加的三级分类数据", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单保存成功",
          type: "success",
        });
        // 关闭对话框
        this.dialogVisible = false;
        // 刷新出新的菜单
        this.getMenus();
        // 设置需要默认展开的菜单
        this.expandedKey = [this.category.parentCid];
      });
    },
    append(node, data) {
      console.log("append", node, data);
      this.dialogType = "add";
      this.title = "添加分类";
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.name = "";
      this.category.catId = null;
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.sort = 0;
      this.category.showStatus = 1;
    },
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单？`, "提示", {
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
            this.$message({
              message: "菜单删除成功",
              type: "success",
            });
            // 刷新出新的菜单
            this.getMenus();
            // 设置需要默认展开的菜单
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {});
      console.log("remove", node, data);
    },
  },
  created() {
    this.getMenus();
  },
  mounted() {},
  beforeCreate() {},
  beforeMount() {},
  beforeUpdate() {},
  updated() {},
  beforeDestroy() {},
  destroyed() {},
  activated() {},
};
</script>
<style scoped>
</style>