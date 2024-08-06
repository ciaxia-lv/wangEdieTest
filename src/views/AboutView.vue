<template>
  <div class="about" ref="aboutImg" >
    <div class="about-header"><button>图片展示</button><button @click="goHome">返回首页</button> </div>
    <div class="img-container">
      <img :src="currentImage" :alt="'Image ' + (currentIndex + 1)" class="img-item">
    </div>
    <button @click="prevImage" class="pre-button">上一页</button>
    <button @click="nextImage" class="next-button">下一页</button>
  </div>
</template>

<script lang="ts" setup>
import {onMounted, ref} from "vue";
import {useRouter} from "vue-router";
const router = useRouter()
let aboutImg = ref(null)
// onMounted(()=>{
//   requestScreen()
// })
const images = ["/src/assets/img/img2.jpg", "/src/assets/img/img3.jpg","/src/assets/img/img4.jpg", "/src/assets/img/img5.jpg"]

// 请求全屏的方法
const requestScreen = () => {
  if (aboutImg.value && !document.fullscreenElement) {
    aboutImg.value.requestFullscreen();
  }
};
// requestScreen()
let currentImage = ref("/src/assets/img/img1.jpeg");
let currentIndex = ref(0)
const prevImage = ()=>{
  if (currentIndex.value > 0) {
    currentIndex.value--;
    currentImage.value = images[currentIndex.value];
  }
}
const nextImage = ()=>{
  if (currentIndex.value < images.length - 1) {
    currentIndex.value++;
    currentImage.value = images[currentIndex.value];
  }
}
const goHome = () => {
  router.push({
    name:"home"
  })
}
</script>

<style lang="less" scoped>
.about-header{
  display: flex;
  justify-content: space-between;
}
.pre-button{
  position: absolute;
  left: 20px;
  top:50%
}
.next-button{
  position: absolute;
  right: 20px;
  top:50%
}
.img-container{
  width: 100%;
  height: 500px;
  overflow: hidden;
}
.img-item, .img-item-container{
  width: 100%;
  height: 100%;
}
</style>
