<template>
	<view class="content">
		<view class="example">
			<uni-steps :data="steps" :active="currentSteps - 1"></uni-steps>
			<button type="primary" v-bind:disabled="currentSteps > 1" v-on:click="scanMaterial"><text>扫物料</text></button>
			<button type="primary" v-bind:disabled="currentSteps != 1" v-on:click="scanWarehouse"><text>扫库位</text></button>
			<view v-if="material.id.length > 0">
				<view class="uni-card">
					<view class="uni-card__header">
						<view class="uni-card__header-title-text">{{ material.code }}</view>
						<view class="uni-card__header-extra-text">{{ material.totalAmount }}</view>
					</view>
					<view class="uni-card__content uni-card__content--pd">
						<view v-for="item in material.goods" v-bind:key="item" class="wxc-list">
							<view class="wxc-list-title-text">
								{{ material.storage == null ? '正在等待库位码，可继续物料' : '出库货架' }}
								<text style="color: #0FAEFF;margin-left: 4px;" v-if="material.storage != null">{{ material.storage.code }}</text>
							</view>
							<view class="wxc-list-extra-text">{{ item }}</view>
						</view>
					</view>
					<view class="uni-card__footer">
						紧急出库:{{ material.code }}
						<text v-if="material.storage != null">{{ material.storage.code }}</text>
					</view>
				</view>
			</view>
			<button type="primary" v-bind:disabled="!sureOutlibrarys" @click="sureOutlibrary">确认出库</button>
			<button type="primary" v-show="currentSteps == 3" @click="goBack">返回</button>
			<!-- <button type="primary"  @click="logMessage">
				浏览器打印
			</button> -->
		</view>
	</view>
</template>

<script>
import { uniSteps, uniCard, uniList, uniListItem } from '@dcloudio/uni-ui';
import outlibraryModel from '@/model/outlibraryModel.js';
import { addUserParam,authAccount, parseForRule, parseWarehouseCode } from '@/libs/util.js';
import { saveEmergentOutInfo } from '@/api/outlibrary.js';
import { mapState } from 'vuex';
export default {
	data() {
		return {
			material: outlibraryModel,
			currentSteps: 0, //当前执行步骤，
			steps: [
				{
					title: '扫物料码'
				},
				{
					title: '扫库位码'
				},
				{
					title: '出库完成'
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
		sureOutlibrarys() {
			if (this.currentSteps == 2) {
				return true;
			} else {
				return false;
			}
		}
	},
	methods: {
		//扫描物料码
		scanMaterial: function(res) {
			var _this = this;
			uni.scanCode({
				onlyFromCamera: true,
				success: function(res) {
					console.log('res' + JSON.stringify(res));
					var result = parseForRule(res.result);
					console.log('result' + JSON.stringify(result));
					if (result && result.code && result.code != '') {
						if (_this.material.code != '' && result.code != _this.material.code) {
							uni.showModal({
								title: '提示',
								showCancel: false,
								content: '跟前一次物料不一致',
								success: function(res) {
									if (res.confirm) {
										console.log('用户点击确定');
									}
								}
							});
						} else {
							if (_this.material.setMateriaInfo(result)) {
								_this.currentSteps = 1;
							} else {
								uni.showToast({
									icon: 'none',
									duration: 2500,
									title: '物料信息错误:' + JSON.stringify(result)
								});
							}
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
		//扫描库位码
		scanWarehouse: function(res) {
			var _this = this;
			uni.scanCode({
				onlyFromCamera: true,
				success: function(res) {
					console.log('res' + JSON.stringify(res));
					var result = parseWarehouseCode(res.result);
					console.log('result' + JSON.stringify(result));
					if (result) {
						if (_this.material.addStorage(result)) {
							_this.currentSteps = 2;
						} else {
							uni.showToast({
								icon: 'none',
								duration: 2500,
								title: '库位信息错误：' + JSON.stringify(result)
							});
						}
					} else {
						uni.showToast({
							icon: 'none',
							duration: 2500,
							title: '库位信息错误:' + res.result
						});
					}
				}
			});
		},
		//确定出库
		sureOutlibrary: function(res) {
			var _this = this;
			saveEmergentOutInfo(addUserParam(this.material.generateModel(), this.userName, this.password, this.userID)).then(data => {
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
		},
		logMessage: function() {
			debugger;
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
			width: $card-extra-width;
			margin-left: $uni-spacing-col-base;
			font-size: $uni-font-size-base;
			text-align: right;
			@include text-omit;
		}
	}
}
</style>
