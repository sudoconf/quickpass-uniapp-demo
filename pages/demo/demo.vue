<template>
	<view>
		<view><text>提示：</text><text>{{ loginTip }}</text></view>
		<view>token: <text>{{ token }}</text></view>
		<view>accessToken: <text>{{ accessToken }}</text></view>
		<button @click="initLogin">初始化</button>
		<button @click="verifyNumber">授权验证</button>
		<button @click="postToken">服务端验证</button>
	</view>
</template>

<script>
	import iosConfig from './ios-config.json'
	import androidConfig from './config.json'
	
	const platform = uni.getSystemInfoSync().platform
	
	const ydLogin = uni.requireNativePlugin('YD-Login')
	export default {
		data() {
			return {
				token: '',
				accessToken: '',
				loginTip: '',
				step: 0
			}
		},
		methods: {
			initLogin () {
				this.token = ''
				this.accessToken = ''
				// 判断是否可以一键登录
				ydLogin.shouldQuickLogin((data) => {
					if (!data.success) {
						this.loginTip = '请检查网络，必须开启蜂窝流量，必须上网网卡为移动、电信、联通运营商'
						return
					}
					
					ydLogin.registerWithBusinessID({
						businessId: '从易盾获取的id',
						timeout: 3000
					}, (data) => {
						if (data.success) {
							this.loginTip = '初始化成功'
							this.step = 1
						} else {
							this.loginTip = '初始化失败'
						}
					})
				})
			},
			verifyNumber () {
				if (this.step !== 1) {
					uni.showToast({
						icon: 'none',
						title: '请先初始化号码认证',
						duration: 800
					})
					return
				}
				// 自定义授权页视图
				let config = {}
				if (platform === 'android') {
					config = androidConfig
				} else if (platform === 'ios') {
					config = iosConfig
				}
				ydLogin.setCustomView(config, this.handleVerificationEvent)
				
				// 预取号
				ydLogin.getPhoneNumberCompletion((data) => {
					if (!data.success) {
						this.loginTip = data.msg || '预取号失败'
						this.token = ''
						return
					}
					this.loginTip = ''
					this.token = data.token

					// 拉起授权页面
					ydLogin.cucmctAuthorizeLoginCompletion((data) => {
						if (!data.success && !data.cancel) {
							this.loginTip = '授权失败'
							return
						} else if (data.cancel) {
							this.loginTip = '用户取消取号'
							return
						}
						this.loginTip = ''
						this.accessToken = data.accessToken
						ydLogin.closeAuthController()
					})
				})
			},
			handleVerificationEvent (data) {
				let title = ''
				if (platform === 'ios') {
					// * iOS授权页生命周期action:
					// authViewDidLoad -- 加载授权页
					// authViewWillAppear -- 授权页已经出现
					// authViewWillDisappear -- 授权页将要消失
					// authViewDidDisappear -- 授权页已经消失
					// authViewDealloc -- 授权页销毁

					// * 页面默认组件点击事件action
					// loginAction -- 登录按钮点击事件（含checked: 0/1）
					// checkedAction -- 复选框事件（含checked: 0/1）
					// appDPrivacy -- 默认协议点击事件
					// appFPrivacy -- 第一个协议点击事件
					// appSPrivacy -- 第二个协议点击事件
					title = `action: ${data.action};checked: data.checked`
				} else if (platform === 'android') {
					// * android授权页生命周期钩子：lifecycle
					// onCreate -- 页面创建
					// onStart -- 页面已开始活动
					// onResume -- 页面展示
					// onPause -- 页面非活动状态
					// onStop -- 页面已停止
					// onDestroy -- 页面销毁
					
					// * 页面默认组件点击事件: clickViewType
					// privacy -- 隐私协议点击事件
					// checkbox -- 复选框点击事件（含isCheckboxChecked: 0/1）
					// loginButton -- 登录按钮点击事件（含isCheckboxChecked: 0/1）
					// leftBackButton -- 左上角返回按钮点击事件
					if (data.viewId) {
						// 自定义组件事件
						title = `点击: ${data.viewId}`
					} else if (data.lifecycle) {
						title = `周期: ${data.lifecycle}`
					} else {
						title = '未知'
					}
					console.log(data)
				}
				uni.showToast({
					icon: '',
					title: title,
					duration: 800
				})
			},
			postToken () {
				uni.request({
					url: 'https://ye.dun.163yun.com/api/login/oneclick',
					method: 'POST',
					data: {
						accessToken: this.accessToken,
						token: this.token
					},
					success: function (res) {
						if (res.data.code === 200) {
							uni.showToast({
								title: '验证成功',
								icon: 'success'
							})
						}
					}
				})
			}
		}
	}
</script>

<style>

</style>
