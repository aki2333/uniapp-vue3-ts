<script setup lang="ts">
import { getMemberAddressByIdApi, postMemberAddressApi, putMemberAddressByIdApi } from '@/services/address';
import { onLoad } from '@dcloudio/uni-app';
import { ref } from 'vue'

// 表单数据
const form = ref({
  receiver: '', // 收货人
  contact: '', // 联系方式
  fullLocation: '', // 省市区(前端展示)
  provinceCode: '', // 省份编码(后端参数)
  cityCode: '', // 城市编码(后端参数)
  countyCode: '', // 区/县编码(后端参数)
  address: '', // 详细地址
  isDefault: 0, // 默认地址，1为是，0为否
})

const query =defineProps<{
  id: string
}>()
uni.setNavigationBarTitle({
  title: query.id ? '编辑地址' : '新增地址',
})
const getMemberAddressByIdData = async () => {
  // 调用接口获取数据
  const res = await getMemberAddressByIdApi(query.id)
  Object.assign(form.value, res.result)
}
onLoad(() => {
  if (query.id) {
    getMemberAddressByIdData()
  }
})

//收集所在地区数据
const onRegionChange:UniHelper.RegionPickerOnChange=(ev)=>{
  // 省市区(前端展示)
  form.value.fullLocation=ev.detail.value.join(' ')
  // 编码(后端参数)
  const [provinceCode, cityCode, countyCode] = ev.detail.code!
  Object.assign(form.value, { provinceCode, cityCode, countyCode })
}
//收集是否默认地址数据
const onIsDefaultChange:UniHelper.SwitchOnChange=(ev)=>{
  form.value.isDefault=ev.detail.value?1:0
}

//表单实例
const formRef=ref<UniHelper.UniFormsInstance>()
// 表单校验规则
const rules:UniHelper.UniFormsRules={
  receiver:{
    rules:[
      {required:true,errorMessage:'请填写收货人姓名'}
    ]
  },
  contact:{
    rules:[
      {required:true,errorMessage:'请填写收货人手机号码'},
      {pattern:/^1[3-9]\d{9}$/,errorMessage:'请填写正确的手机号码'}
    ]
  },
  fullLocation:{
    rules:[
      {required:true,errorMessage:'请选择省/市/区(县)'}
    ]
  },
  address:{
    rules:[
      {required:true,errorMessage:'请填写详细地址'}
    ]
  }
}

//提交表单
const onSubmit=async()=>{
  try {
    const res= await formRef.value?.validate?.()
    console.log(res);
    
    if(query.id){
      // 编辑地址
      await putMemberAddressByIdApi(query.id,form.value)
    }else{
    await postMemberAddressApi(form.value)
    }
    uni.showToast({
      title: query.id?'修改成功':'保存成功',
      icon: 'success',
    })
    setTimeout(() => {
      uni.navigateBack()
    }, 500)
  } catch (error) {
    uni.showToast({
      title: '请检查表单是否填写完整',
      icon: 'none',
    })
  }

}
</script>

<template>
  <view class="content">
    <uni-forms :rules="rules" ref="formRef" :model="form">
      <!-- 表单内容 -->
      <uni-forms-item name="receiver" class="form-item">
        <text class="label">收货人</text>
        <input class="input" placeholder="请填写收货人姓名" v-model="form.receiver" />
      </uni-forms-item>
      <uni-forms-item name="contact" class="form-item">
        <text class="label">手机号码</text>
        <input class="input" placeholder="请填写收货人手机号码" v-model="form.contact" />
      </uni-forms-item>
      <uni-forms-item name="fullLocation" class="form-item">
        <text class="label">所在地区</text>
        <picker @change="onRegionChange" class="picker" mode="region" :value="form.fullLocation.split(' ')">
          <view v-if="form.fullLocation">{{ form.fullLocation }}</view>
          <view v-else class="placeholder">请选择省/市/区(县)</view>
        </picker>
      </uni-forms-item>
      <uni-forms-item name="address" class="form-item">
        <text class="label">详细地址</text>
        <input class="input" placeholder="街道、楼牌号等信息" v-model="form.address" />
      </uni-forms-item>
      <view class="form-item">
        <label class="label">设为默认地址</label>
        <switch @change="onIsDefaultChange" class="switch" color="#27ba9b" :checked="form.isDefault===1" />
      </view>
    </uni-forms>
  </view>
  <!-- 提交按钮 -->
  <button class="button" @tap="onSubmit">保存并使用</button>
</template>

<style lang="scss">
page {
  background-color: #f4f4f4;
}

.content {
  margin: 20rpx 20rpx 0;
  padding: 0 20rpx;
  border-radius: 10rpx;
  background-color: #fff;

  .form-item,
  .uni-forms-item {
    display: flex;
    align-items: center;
    min-height: 96rpx;
    padding: 25rpx 10rpx 40rpx;
    background-color: #fff;
    font-size: 28rpx;
    border-bottom: 1rpx solid #ddd;
    position: relative;
    margin-bottom: 0;

    // 调整 uni-forms 样式
    .uni-forms-item__content {
      display: flex;
    }

    .uni-forms-item__error {
      margin-left: 200rpx;
    }

    &:last-child {
      border: none;
    }

    .label {
      width: 200rpx;
      color: #333;
    }

    .input {
      flex: 1;
      display: block;
      height: 46rpx;
    }

    .switch {
      position: absolute;
      right: -20rpx;
      transform: scale(0.8);
    }

    .picker {
      flex: 1;
    }

    .placeholder {
      color: #808080;
    }
  }
}

.button {
  height: 80rpx;
  margin: 30rpx 20rpx;
  color: #fff;
  border-radius: 80rpx;
  font-size: 30rpx;
  background-color: #27ba9b;
}
</style>