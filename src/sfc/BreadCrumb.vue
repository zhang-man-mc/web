<!-- 面包屑 -->
<script>
import { ArrowRight } from '@element-plus/icons-vue'
export default {
   data(){
      return{
        list:[],
        subTitle1: null,
        subTitle2 :null,
      }
   },
 setup(){
    return {ArrowRight}
 },
   watch:{
    $route(to,from){
        this.list = to.matched
        // this.subTitle1 = this.list[1]
        // this.subTitle2 = this.list[2]
    }
   },
   mounted (){
    this.list = this.$route.matched
    // this.subTitle1 = this.list[1]
    // this.subTitle2 = this.list[2]
   },
   methods:{
   }
}
</script>

<template>
    <el-breadcrumb :separator-icon="ArrowRight">
            <el-breadcrumb-item v-for="item in list" :key="item.path" :to="{ path: '/' }">
                <router-link :to="item.path">{{ item.meta.title }}</router-link>
            </el-breadcrumb-item>
    </el-breadcrumb>

    <!-- <el-breadcrumb :separator-icon="ArrowRight">
      <el-breadcrumb-item>  {{subTitle1.meta.title}} </el-breadcrumb-item>
      <el-breadcrumb-item>  <router-link :to="item.path">{{subTitle2.meta.title}}</router-link></el-breadcrumb-item> -->

   

    
</template>
<style scoped>
.el-breadcrumb__inner a {
  font-weight: 500;
  font-size: 16px;
  color: rgba(0, 0, 0);
}

.el-breadcrumb__item:last-child .el-breadcrumb__inner a {
    color: rgb(209, 207, 207);
}
</style>