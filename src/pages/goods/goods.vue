<script setup lang="ts">
import { getGoodsByIdApi } from '@/services/goods';
import type { GoodsResult } from '@/types/goods';
import { onLoad } from '@dcloudio/uni-app';
import { computed, ref } from 'vue';
import AddressPanel from './components/AddressPanel.vue';
import ServicePanel from './components/ServicePanel.vue';
import type { SkuPopupEvent, SkuPopupInstance, SkuPopupLocaldata } from '@/components/vk-data-goods-sku-popup/vk-data-goods-sku-popup';
import { postMemberCartApi } from '@/services/cart';
import { useAddressStore } from '@/stores/modules/address';

//获取屏幕边界到安全区域距离
const { safeAreaInsets } = uni.getSystemInfoSync()
//获取传参商品id
const query = defineProps<{ id: string }>()
// 获取商品详情数据
const goods = ref<GoodsResult>()
const getGoodsByIdData = async () => {
  const res = await getGoodsByIdApi(query.id)
  goods.value = res.result
  //封装 sku组件所需数据
  localdata.value = {
    _id:res.result.id,
    name:res.result.name,
    goods_thumb:res.result.mainPictures[0],
    spec_list:res.result.specs.map(s=>({name:s.name,list:s.values})),
    sku_list:res.result.skus.map(v=>({
      _id:v.id,
      goods_id:res.result.id,
      goods_name:res.result.name,
      image:v.picture,
      price:v.price*100,
      sku_name_arr:v.specs.map(s=>s.valueName),
      stock:v.inventory
    }))
  }
}

onLoad(() => {
  getGoodsByIdData()
})
const currentIndex = ref(0)
const onChange: UniHelper.SwiperOnChange = (ev) => {
  currentIndex.value = ev.detail.current
}
const onTapImage = (url: string) => {
  uni.previewImage({
    urls: goods.value?.mainPictures || [],
    current: url
  })
}
// 弹出层
const popup = ref<{
  open: (type?: UniHelper.UniPopupType) => void,
  close: () => void
}>()
const popupType = ref<'address' | 'service'>()
const onPopupTap = (type: typeof popupType.value) => {
  popupType.value = type
  popup.value?.open()
}
// sku
const isShowSku = ref(false)
const skuPopup = ref<SkuPopupInstance>()
const localdata = ref<SkuPopupLocaldata>()
enum SkuMode {
  Both = 1,
  AddCart = 2,
  BuyNow = 3
}
const mode=ref<SkuMode>(SkuMode.Both)
const onSkuTap = (val:SkuMode) => {
  mode.value=val
  isShowSku.value = true
}
const selectArrText=computed(()=>{
  return skuPopup.value?.selectArr?.join(' ').trim()||'请选择商品规格'
})
const addressStore= useAddressStore()
const userAddresses=computed(()=>{
  return addressStore.selectedAddress?.fullLocation||
  // goods.value?.userAddresses.find(item=>item.isDefault)?.fullLocation||
  '请选择收获地址'
})

//加入购物车
const onAddCart= async(ev:SkuPopupEvent)=>{
  await postMemberCartApi({ skuId: ev._id, count: ev.buy_num })
  uni.showToast({
    title:'加入购物车成功',
    icon:'success'
  })
  isShowSku.value=false
}
//立即购买
const onBuyNow= async(ev:SkuPopupEvent)=>{
  uni.navigateTo({
    url:`/pagesOrder/create/create?skuId=${ev._id}&count=${ev.buy_num}`
  })
}
</script>

<template>
  <scroll-view scroll-y class="viewport">
    <!-- 基本信息 -->
    <view class="goods">
      <!-- 商品主图 -->
      <view class="preview">
        <swiper circular @change="onChange">
          <swiper-item v-for="item in goods?.mainPictures" :key="item">
            <image mode="aspectFill" :src="item" @tap="onTapImage(item)" />
          </swiper-item>

        </swiper>
        <view class="indicator">
          <text class="current">{{ currentIndex + 1 }}</text>
          <text class="split">/</text>
          <text class="total">{{ goods?.mainPictures.length }}</text>
        </view>
      </view>

      <!-- 商品简介 -->
      <view class="meta">
        <view class="price">
          <text class="symbol">¥</text>
          <text class="number">{{ goods?.price }}</text>
        </view>
        <view class="name ellipsis">{{ goods?.name }} </view>
        <view class="desc"> {{ goods?.desc }} </view>
      </view>

      <!-- 操作面板 -->
      <view class="action">
        <view class="item arrow" @tap="onSkuTap(SkuMode.Both)">
          <text class="label">选择</text>
          <text class="text ellipsis"> {{ selectArrText }} </text>
        </view>
        <view class="item arrow" @tap="onPopupTap('address')">
          <text class="label">送至</text>
          <text class="text ellipsis">{{ userAddresses }}  </text>
        </view>
        <view class="item arrow" @tap="onPopupTap('service')">
          <text class="label">服务</text>
          <text class="text ellipsis"> 无忧退 快速退款 免费包邮 </text>
        </view>
      </view>
    </view>

    <!-- 商品详情 -->
    <view class="detail panel">
      <view class="title">
        <text>详情</text>
      </view>
      <view class="content">
        <view class="properties">
          <!-- 属性详情 -->
          <view class="item" v-for="item in goods?.details.properties" :key="item.name">
            <text class="label">{{ item.name }}</text>
            <text class="value">{{ item.value }}</text>
          </view>
        </view>
        <!-- 图片详情 -->
        <image v-for="item in goods?.details.pictures" :key="item" mode="widthFix" :src="item"></image>

      </view>
    </view>

    <!-- 同类推荐 -->
    <view class="similar panel">
      <view class="title">
        <text>同类推荐</text>
      </view>
      <view class="content">
        <navigator v-for="item in goods?.similarProducts" :key="item.id" class="goods" hover-class="none"
          :url="`/pages/goods/goods?id=${item.id}`">
          <image class="image" mode="aspectFill" :src="item.picture"></image>
          <view class="name ellipsis">{{ item.name }}</view>
          <view class="price">
            <text class="symbol">¥</text>
            <text class="number">{{ item.price }}</text>
          </view>
        </navigator>
      </view>
    </view>
  </scroll-view>

  <!-- 用户操作 -->
  <view class="toolbar" :style="{ paddingBottom: safeAreaInsets?.bottom + 'px' }">
    <view class="icons">
      <button class="icons-button"><text class="icon-heart"></text>收藏</button>
      <!-- #ifdef MP-WEIXIN -->
      <button class="icons-button" open-type="contact">
        <text class="icon-handset"></text>客服
      </button>
      <!-- #endif -->
      <navigator class="icons-button" url="/pages/cart/cart2">
        <text class="icon-cart"></text>购物车
      </navigator>
    </view>
    <view class="buttons">
      <view class="addcart" @tap="onSkuTap(SkuMode.AddCart)"> 加入购物车 </view>
      <view class="buynow" @tap="onSkuTap(SkuMode.BuyNow)"> 立即购买 </view>
    </view>
  </view>

  <!-- 弹出层 -->
  <uni-popup ref="popup" type="bottom" background-color="#ffffff">
    <AddressPanel v-if="popupType === 'address'" @close="popup?.close" />
    <ServicePanel v-if="popupType === 'service'" @close="popup?.close" />
  </uni-popup>

  <!-- sku -->
  <vk-data-goods-sku-popup
   ref="skuPopup"
   v-model="isShowSku"
   :localdata="localdata"
   :mode="mode"
   buy-now-background-color="#28BA9B"
   add-cart-background-color="#FFA868"
   :actived-style="{
      color: '#28BA9B',
      borderColor: '#28BA9B',
      backgroundColor: '#E9F8F5'
   }"
   @add-cart="onAddCart"
   @buy-now="onBuyNow"
   ></vk-data-goods-sku-popup>
</template>

<style lang="scss">
page {
  height: 100%;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.viewport {
  background-color: #f4f4f4;
}

.panel {
  margin-top: 20rpx;
  background-color: #fff;

  .title {
    display: flex;
    justify-content: space-between;
    align-items: center;
    height: 90rpx;
    line-height: 1;
    padding: 30rpx 60rpx 30rpx 6rpx;
    position: relative;

    text {
      padding-left: 10rpx;
      font-size: 28rpx;
      color: #333;
      font-weight: 600;
      border-left: 4rpx solid #27ba9b;
    }

    navigator {
      font-size: 24rpx;
      color: #666;
    }
  }
}

.arrow {
  &::after {
    position: absolute;
    top: 50%;
    right: 30rpx;
    content: '\e6c2';
    color: #ccc;
    font-family: 'erabbit' !important;
    font-size: 32rpx;
    transform: translateY(-50%);
  }
}

/* 商品信息 */
.goods {
  background-color: #fff;

  .preview {
    height: 750rpx;
    position: relative;

    .image {
      width: 750rpx;
      height: 750rpx;
    }

    .indicator {
      height: 40rpx;
      padding: 0 24rpx;
      line-height: 40rpx;
      border-radius: 30rpx;
      color: #fff;
      font-family: Arial, Helvetica, sans-serif;
      background-color: rgba(0, 0, 0, 0.3);
      position: absolute;
      bottom: 30rpx;
      right: 30rpx;

      .current {
        font-size: 26rpx;
      }

      .split {
        font-size: 24rpx;
        margin: 0 1rpx 0 2rpx;
      }

      .total {
        font-size: 24rpx;
      }
    }
  }

  .meta {
    position: relative;
    border-bottom: 1rpx solid #eaeaea;

    .price {
      height: 130rpx;
      padding: 25rpx 30rpx 0;
      color: #fff;
      font-size: 34rpx;
      box-sizing: border-box;
      background-color: #35c8a9;
    }

    .number {
      font-size: 56rpx;
    }

    .brand {
      width: 160rpx;
      height: 80rpx;
      overflow: hidden;
      position: absolute;
      top: 26rpx;
      right: 30rpx;
    }

    .name {
      max-height: 88rpx;
      line-height: 1.4;
      margin: 20rpx;
      font-size: 32rpx;
      color: #333;
    }

    .desc {
      line-height: 1;
      padding: 0 20rpx 30rpx;
      font-size: 24rpx;
      color: #cf4444;
    }
  }

  .action {
    padding-left: 20rpx;

    .item {
      height: 90rpx;
      padding-right: 60rpx;
      border-bottom: 1rpx solid #eaeaea;
      font-size: 26rpx;
      color: #333;
      position: relative;
      display: flex;
      align-items: center;

      &:last-child {
        border-bottom: 0 none;
      }
    }

    .label {
      width: 60rpx;
      color: #898b94;
      margin: 0 16rpx 0 10rpx;
    }

    .text {
      flex: 1;
      -webkit-line-clamp: 1;
    }
  }
}

/* 商品详情 */
.detail {
  padding-left: 20rpx;

  .content {
    margin-left: -20rpx;

    .image {
      width: 100%;
    }
  }

  .properties {
    padding: 0 20rpx;
    margin-bottom: 30rpx;

    .item {
      display: flex;
      line-height: 2;
      padding: 10rpx;
      font-size: 26rpx;
      color: #333;
      border-bottom: 1rpx dashed #ccc;
    }

    .label {
      width: 200rpx;
    }

    .value {
      flex: 1;
    }
  }
}

/* 同类推荐 */
.similar {
  .content {
    padding: 0 20rpx 200rpx;
    background-color: #f4f4f4;
    display: flex;
    flex-wrap: wrap;

    .goods {
      width: 340rpx;
      padding: 24rpx 20rpx 20rpx;
      margin: 20rpx 7rpx;
      border-radius: 10rpx;
      background-color: #fff;
    }

    .image {
      width: 300rpx;
      height: 260rpx;
    }

    .name {
      height: 80rpx;
      margin: 10rpx 0;
      font-size: 26rpx;
      color: #262626;
    }

    .price {
      line-height: 1;
      font-size: 20rpx;
      color: #cf4444;
    }

    .number {
      font-size: 26rpx;
      margin-left: 2rpx;
    }
  }

  navigator {
    &:nth-child(even) {
      margin-right: 0;
    }
  }
}

/* 底部工具栏 */
.toolbar {
  position: fixed;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 1;
  background-color: #fff;
  height: 100rpx;
  padding: 0 20rpx var(--window-bottom);
  border-top: 1rpx solid #eaeaea;
  display: flex;
  justify-content: space-between;
  align-items: center;
  box-sizing: content-box;

  .buttons {
    display: flex;

    &>view {
      width: 220rpx;
      text-align: center;
      line-height: 72rpx;
      font-size: 26rpx;
      color: #fff;
      border-radius: 72rpx;
    }

    .addcart {
      background-color: #ffa868;
    }

    .buynow,
    .payment {
      background-color: #27ba9b;
      margin-left: 20rpx;
    }
  }

  .icons {
    padding-right: 10rpx;
    display: flex;
    align-items: center;
    flex: 1;

    .icons-button {
      flex: 1;
      text-align: center;
      line-height: 1.4;
      padding: 0;
      margin: 0;
      border-radius: 0;
      font-size: 20rpx;
      color: #333;
      background-color: #fff;

      &::after {
        border: none;
      }
    }

    text {
      display: block;
      font-size: 34rpx;
    }
  }
}
</style>