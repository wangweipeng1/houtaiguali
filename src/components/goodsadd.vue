<template>
  <el-card class="box">
    <cus-bread leave1="商品管理" leave2="商品列表"></cus-bread>
    <el-alert class="alert" title="添加商品信息" type="info" center show-icon :closable='false'></el-alert>
    <el-steps :active="active*1" align-center finish-status="success">
      <el-step title="基本信息"></el-step>
      <el-step title="商品参数"></el-step>
      <el-step title="商品属性"></el-step>
      <el-step title="商品图片"></el-step>
      <el-step title="商品内容"></el-step>
    </el-steps>
    <el-form class="form" label-position="top" label-width="80px" :model="form">
      <el-tabs @tab-click="changeTab()" v-model="active" tab-position="left">
        <el-tab-pane name="1" label="基本信息">
          <el-form-item label="商品名称">
            <el-input v-model="form.goods_name"></el-input>
          </el-form-item>
          <el-form-item label="商品价格">
            <el-input v-model="form.goods_price"></el-input>
          </el-form-item>
          <el-form-item label="商品重量">
            <el-input v-model="form.goods_weight"></el-input>
          </el-form-item>
          <el-form-item label="商品数量">
            <el-input v-model="form.goods_number"></el-input>
          </el-form-item>
          <el-form-item label="商品分类">
            <el-cascader
              expand-trigger="hover"
              :options="options"
              v-model="selectedOptions"
              @change="handleChange"
              :props="defaultProp"
              clearable
            ></el-cascader>
          </el-form-item>
        </el-tab-pane>
        <el-tab-pane name="2" label="商品参数">
          <el-form-item :label="item1.attr_name" v-for="(item1, i) in arrDy" :key="item1.attr_id">
            <el-checkbox-group v-model="item1.attr_vals">
              <el-checkbox border :label="item2" v-for="(item2, i) in item1.attr_vals" :key="i"></el-checkbox>
            </el-checkbox-group>
          </el-form-item>
        </el-tab-pane>
        <el-tab-pane name="3" label="商品属性">
          <el-form-item :label="item.attr_name" v-for="(item, i) in arrStatic" :key="item.attr_id">
            <el-input v-model="item.attr_vals"></el-input>
          </el-form-item>
        </el-tab-pane>
        <el-tab-pane name="4" label="商品图片">
          <el-form-item label="添加图片">
            <el-upload
              :headers="headers"
              action="http://localhost:8888/api/private/v1/upload"
              :on-success="handleSuccess"
              :on-remove="handleRemove"
              list-type="picture"
            >
              <el-button size="small" type="primary">点击上传</el-button>
            </el-upload>
          </el-form-item>
        </el-tab-pane>
        <el-tab-pane name="5" label="商品内容">
          <el-form-item>
            <el-button @click="addGoods()">添加商品</el-button>
            <quill-editor v-model="form.goods_introduce"></quill-editor>
          </el-form-item>
        </el-tab-pane>
      </el-tabs>
    </el-form>
  </el-card>
</template>

<script>
// 富文本编辑器
import "quill/dist/quill.core.css";
import "quill/dist/quill.snow.css";
import "quill/dist/quill.bubble.css";
import { quillEditor } from "vue-quill-editor";
export default {
  components: { quillEditor },
  data() {
    return {
      active: "1",
      form: {
        goods_name: "",
        goods_cat: "",
        goods_price: "",
        goods_number: "",
        goods_weight: "",
        goods_introduce: "",
        pics: [],
        attrs: []
      },
      //   级联配置数据源
      options: [],
      selectedOptions: [],
      defaultProp: {
        label: "cat_name",
        value: "cat_id"
        // children: "children"
      },
      arrDy: [],
      arrStatic: [],
      fileList2: [],
      // 上传文件
      headers: {
        Authorization: localStorage.getItem("token")
      }
    };
  },
  created() {
    this.getGoodsCate();
  },
  methods: {
    // 添加商品请求
    async addGoods() {
      this.form.goods_cat = this.selectedOptions.join(",");
      // 处理动态;
      const obj1 = { attr_id: "", attr_value: "" };
      const arr1 = [];
      this.arrDy.forEach(item => {
        obj1.attr_id = item.attr_id;
        obj1.attr_value = item.attr_vals;
        arr1.push(obj1);
      });

      // 处理静态;
      const obj2 = { attr_id: "", attr_value: "" };
      const arr2 = [];
      this.arrStatic.forEach(item => {
        obj2.attr_id = item.attr_id;
        obj2.attr_value = item.attr_vals;
        arr2.push(obj2);
      });
      this.form.attrs = [...arr1, ...arr2];

      const res = await this.$http.post("goods", this.form);
      const {
        meta: { msg, status }
      } = res.data;
      if (status === 201) {
        // 列表
        this.$router.push({
          name: "goods"
        });
      } else {
        this.$message.error(msg);
      }
    },
    handleSuccess(res, file, fileList) {
      console.log(res, "success--");
      // res.data.tmp_path
      this.form.pics.push({
        pic: res.data.tmp_path
      });
    },
    handleRemove(file, fileList) {
      console.log(file, "remove----");
      // file.response.data.tmp_path
      const Index = this.form.pics.findIndex(item => {
        return item.pic === file.response.data.tmp_path;
      });
      this.form.pics.splice(Index, 1);
    },
    async changeTab() {
      if ((this.active === "2") | (this.active === "3")) {
        if (this.selectedOptions.length !== 3) {
          this.$message.error("请先选择三级分类!");
          if (this.active === "2") {
            this.arrDy = [];
          } else {
            this.arrStatic = [];
          }
          return;
        }
        if (this.active === "2") {
          const res = await this.$http.get(
            `categories/${this.selectedOptions[2]}/attributes?sel=many`
          );
          console.log(res, "获取动态分类参数列表");
          const {
            meta: { status, msg },
            data
          } = res.data;
          if (status === 200) {
            this.arrDy = data;
            this.arrDy.forEach(item => {
              item.attr_vals =
                item.attr_vals.trim().length === 0
                  ? []
                  : item.attr_vals.trim().split(",");
            });
            console.log(this.arrDy, "动态分类参数");
          }
        }
        if (this.active === "3") {
          const res = await this.$http.get(
            `categories/${this.selectedOptions[2]}/attributes?sel=only`
          );
          console.log(res, "获取静态分类参数列表");
          const {
            meta: { status, msg },
            data
          } = res.data;
          if (status === 200) {
            this.arrStatic = data;
            console.log(this.arrStatic, "静态分类参数");
          }
        }
      }
    },
    //   获取商品数据列表
    async getGoodsCate() {
      const res = await this.$http.get(`categories?type=3`);
      console.log(res, "获取商品数据列表");
      const {
        meta: { status, msg },
        data
      } = res.data;
      if (status === 200) {
        this.options = data;
      }
    },
    handleChange() {}
  }
};
</script>

<style scoped>
.box {
  height: 100%;
}
.alert {
  margin-top: 20px;
  margin-bottom: 20px;
}
.form {
  height: 300px;
  overflow: auto;
}
.ql-editor,
.ql-blank {
  /* height: 200px; */
  min-height: 200px;
}
</style>
