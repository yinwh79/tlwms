<template>
	<view class="content">
		<view class="example">
			<uni-steps :data="steps" :active="currentSteps - 1"></uni-steps>
			<button type="primary" v-on:click="scanPackege" v-bind:disabled="currentSteps != 0"><text>扫拣货码</text></button>
			<button type="primary" v-bind:disabled="currentSteps != 1" v-on:click="scanMaterial"><text>扫物料码</text></button>
			<view v-if="material.TlJpdID.length > 0">
				<view class="uni-card">
					<view class="">
						<view class="wxc-list-extra">需求单号:{{ material.OperBillNum }}</view>
						<view class="wxc-list-extra">条码内容:{{ material.BillNum }}</view>
						<view class="wxc-list-extra">物料编码:{{ material.MNumber }}</view>
						<view class="wxc-list-extra">物料名称:{{ material.MName }}</view>
						<view class="wxc-list-extra">包装数量:{{ material.OutPackage }}</view>
						<view class="wxc-list-extra">对应数量:{{ material.Qty }}</view>
						<view class="wxc-list-extra">出库库位:{{ material.LocalName }}</view>
					</view>
				</view>
				<view class="uni-card" v-if="material.id.length > 0">
					<view class="">
						<view class="wxc-list-extra">物料编码:{{ material.code }}</view>
						<view class="wxc-list-extra">物料标志:{{ material.codeFlag }}</view>
					</view>
				</view>
			</view>
			<button type="primary" v-bind:disabled="!isCanOut" @click="sureOutlibrary">确认出库</button>
			<!-- <button type="default" v-bind:disabled="!isReseatPage" @click="reset">返回扫描</button>
			<button type="default" v-show="currentSteps == 3" @click="goBack">返回</button> -->
		</view>
	</view>
</template>

<script>
import { uniSteps, uniCard, uniList, uniListItem } from '@dcloudio/uni-ui';
import { getPickGoodsCodeInfo, surePmkStockOut } from '@/api/outlibrary.js';
import outlibraryModel from '@/model/outlibraryPmkModel.js';
import { mapState } from 'vuex';
import { addUserParam, authAccount, parseForRule, isEmptyObject } from '@/libs/util.js';
export default {
	data() {
		return {
			material: outlibraryModel,
			currentSteps: 0, //当前执行步骤，
			steps: [
				{
					title: '扫拣货码'
				},
				{
					title: '扫物料码'
				},
				{
					title: '确认出库'
				}
			]
		};
	},
	created() {
		this.currentSteps = 0;
		this.material.reset();
	},
	components: {
		uniSteps,
		uniCard,
		uniList,
		uniListItem
	},
	computed: {
		...mapState(['forcedLogin', 'hasLogin', 'userName', 'password', 'userID']),
		isCanOut() {
			if (this.currentSteps == 2) {
				return true;
			} else {
				return false;
			}
		},
		isReseatPage() {
			if (this.currentSteps == 2) {
				return true;
			} else {
				return false;
			}
		}
	},
	methods: {
		reset: function() {
			this.currentSteps = 0;
			this.material.reset();
		},
		//扫描拣货码
		scanPackege: function(res) {
			// if (this.currentSteps >= 1) {
			// 	this.resetScanPackege();
			// } else {
			var _this = this;
			uni.scanCode({
				onlyFromCamera: true,
				success: function(res) {
					console.log('res' + JSON.stringify(res));
					if (res && res.result && res.result != '' && res.result.indexOf('PGC') != '-1') {
						var code = res.result;
						getPickGoodsCodeInfo(res.result, _this.userName, _this.password, _this.userID).then(data => {
							var [error, res] = data;
							console.log('getPickGoodsCodeInfo.data:' + JSON.stringify(data));
							console.log('getPickGoodsCodeInfo.res:' + JSON.stringify(res));
							var result = parseForRule(res.data);
							console.log('result:' + JSON.stringify(result));
							if (result && !isEmptyObject(result)) {
								// if (result.Status && result.Status == '已出库') {
								// 	uni.showModal({
								// 		title: '提示',
								// 		content: '拣货码已经出库',
								// 		showCancel: false,
								// 		success: function(res) {
								// 			if (res.confirm) {
								// 				console.log('用户点击确定');
								// 			}
								// 		}
								// 	});
								// } else {
								_this.currentSteps = 1;
								_this.material.setPackege(result);
								//}
							} else {
								uni.showModal({
									title: '提示',
									content: '没有获取到拣货码信息，请检查拣货码',
									showCancel: false,
									success: function(res) {
										if (res.confirm) {
											console.log('用户点击确定');
										}
									}
								});
							}
						});
					} else {
						uni.showToast({
							icon: 'none',
							duration: 2500,
							title: '拣货码错误,请重新扫描；'
						});
					}
				}
			});
			//}
		},
		resetScanPackege: function(res) {
			var _this = this;
			uni.showModal({
				title: '提示',
				content: '是否放弃当前拣货码，重新扫描拣货码',
				success: function(res) {
					if (res.confirm) {
						_this.reset();
						uni.scanCode({
							onlyFromCamera: true,
							success: function(res) {
								console.log('res' + JSON.stringify(res));
								if (res && res.result && res.result != '' && res.result.indexOf('PGC') != '-1') {
									getPickGoodsCodeInfo(res.result, _this.userName, _this.password, _this.userID).then(data => {
										var [error, res] = data;
										console.log('getPickGoodsCodeInfo.data:' + JSON.stringify(data));
										console.log('getPickGoodsCodeInfo.res:' + JSON.stringify(res));
										var result = parseForRule(res.data);
										console.log('result:' + JSON.stringify(result));
										if (result && !isEmptyObject(result)) {
											if (result.Status && result.Status == '已出库') {
												uni.showModal({
													title: '提示',
													content: '拣货码已经出库',
													showCancel: false,
													success: function(res) {
														if (res.confirm) {
															console.log('用户点击确定');
														}
													}
												});
											} else {
												_this.currentSteps = 1;
												_this.material.setPackege(result);
											}
										} else {
											uni.showModal({
												title: '提示',
												content: '没有获取到拣货码信息，请检查拣货码',
												showCancel: false,
												success: function(res) {
													if (res.confirm) {
														console.log('用户点击确定');
													}
												}
											});
										}
									});
								} else {
									uni.showToast({
										icon: 'none',
										duration: 2500,
										title: '拣货码错误,请重新扫描；'
									});
								}
							}
						});
					} else if (res.cancel) {
					}
				}
			});
		},

		//扫描物料码
		// { id: 'W', code: '1001030001-B12', codeid: '1', count: 12 }
		scanMaterial: function(res) {
			var _this = this;
			uni.scanCode({
				onlyFromCamera: true,
				success: function(res) {
					console.log('res' + JSON.stringify(res));
					var result = parseForRule(res.result);
					console.log('result' + JSON.stringify(result));
					if (result && result.code && result.code != '') {
						if (_this.material.setMateriaInfo(result)) {
							_this.currentSteps = 2;
						} else {
							uni.showToast({
								icon: 'none',
								duration: 2500,
								title: '物料信息错误:' + JSON.stringify(result)
							});
						}
					} else {
						uni.showToast({
							icon: 'none',
							duration: 2500,
							title: '物料信息错误:' + res.result
						});
					}
				}
			});
		},
		//确定生成发车单
		sureOutlibrary: function(res) {
			var _this = this;
			surePmkStockOut(addUserParam(this.material.generateModel(), this.userName, this.password, this.userID)).then(data => {
				var [error, res] = data;
				console.log('data:' + JSON.stringify(data));
				console.log('res:' + JSON.stringify(res));
				var result = parseForRule(res.data);
				console.log('result:' + JSON.stringify(result));
				if (result.success) {
					console.log(result);
					_this.currentSteps = 3;
					console.log('正确');
					uni.showToast({
						icon: 'success',
						title: '出库成功！'
					});
					_this.reset();
				} else {
					console.log('错误');
					uni.showModal({
						title: '提示',
						showCancel: false,
						content: result.ResponseText,
						success: function(res) {
							if (res.confirm) {
								console.log('用户点击确定');
							}
						}
					});
				}
			});
		},
		//返回
		goBack: function() {
			uni.navigateBack();
		}
	},
	onLoad() {
		authAccount(this.hasLogin, this.forcedLogin, this.userName);
	}
};
</script>
<style lang="scss">
.materialnumber {
	width: auto;
	height: 100%;
	float: right;
}

button {
	margin-top: 10px;
}

.bank {
	width: auto;
	height: 100%;
	margin: 0 60%;
	position: absolute;
}

$card-extra-width: 30%;

@mixin text-omit {
	text-overflow: ellipsis;
	white-space: nowrap;
	overflow: hidden;
}

.uni-card {
	margin: $uni-spacing-col-base;
	background: $uni-bg-color;
	position: relative;
	display: flex;
	flex-direction: column;

	&:after {
		content: '';
		position: absolute;
		transform-origin: center;
		box-sizing: border-box;
		pointer-events: none;
		top: -50%;
		left: -50%;
		right: -50%;
		bottom: -50%;
		border: 1px solid $uni-border-color;
		border-radius: $uni-border-radius-lg;
		transform: scale(0.5);
	}

	&__footer,
	&__header {
		position: relative;
		display: flex;
		flex-direction: row;
		padding: $uni-spacing-col-base;
		align-items: center;
	}

	&__header {
		&:after {
			position: absolute;
			bottom: 0;
			right: 0;
			left: 0;
			height: 1px;
			content: '';
			-webkit-transform: scaleY(0.5);
			transform: scaleY(0.5);
			background-color: $uni-border-color;
		}

		&-title {
			flex: 1;
			margin-right: $uni-spacing-col-base;
			display: flex;
			flex-direction: row;
			justify-content: flex-start;
			align-items: center;

			&-text {
				font-size: $uni-font-size-lg;
				flex: 1;
				@include text-omit;
			}
		}

		&-extra {
			&-img-view {
				display: flex;
			}

			&-img {
				height: $uni-img-size-sm;
				width: $uni-img-size-sm;
				margin-right: $uni-spacing-col-base;
			}

			&-text {
				flex: 0 0 auto;
				width: $card-extra-width;
				margin-left: $uni-spacing-col-base;
				font-size: $uni-font-size-base;
				text-align: right;
				@include text-omit;
			}
		}
	}

	&__content {
		&--pd {
			padding: $uni-spacing-col-base;
		}
	}

	&__footer {
		justify-content: space-between;
		color: $uni-text-color-grey;
		font-size: $uni-font-size-sm;
		padding-top: 0;
	}
}

.wxc-list {
	position: relative;
	display: flex;
	flex-direction: row;
	padding: $uni-spacing-col-base;
	padding-right: 0px;
	align-items: center;

	&:after {
		position: absolute;
		bottom: 0;
		right: 0;
		left: 0;
		height: 1px;
		content: '';
		-webkit-transform: scaleY(0.5);
		transform: scaleY(0.5);
		background-color: $uni-border-color;
	}

	&-title {
		flex: 1;
		margin-right: $uni-spacing-col-base;
		display: flex;
		flex-direction: row;
		justify-content: flex-start;
		align-items: center;

		&-text {
			font-size: $uni-font-size-base;
			flex: 1;
			@include text-omit;
		}
	}

	&-extra {
		&-img-view {
			display: flex;
		}

		&-img {
			height: $uni-img-size-sm;
			width: $uni-img-size-sm;
			margin-right: $uni-spacing-col-base;
		}

		&-text {
			flex: 0 0 auto;
			width: 10%;
			margin-left: $uni-spacing-col-base;
			font-size: $uni-font-size-base;
			text-align: right;
			@include text-omit;
		}
	}
}
</style>
