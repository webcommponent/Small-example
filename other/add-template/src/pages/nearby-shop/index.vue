<template>
    <div>
        <div class="no_pos" v-if="noPos">
            <div class="no_pos_con">
                <i class="icon icon_no_pos"></i>
                <p>无法获取地理位置，请选择收货地址</p>
                <div class="search_address" @click="searchAddress">搜索地址</div>
            </div>
        </div>
        <div class="nearby_shop" v-if="block">
            <div class="nearby_top">
                <div class="search" @click="search">
                    <i class="icon icon_sear"></i>
                    <p class="text">搜索</p>
                </div>
            </div>
            <!-- <scroll-view scroll-y="true" :style="{height:winHeight+'px'}" lower-threshold="20" @scrolltolower="scrollHandler"> -->
            <div class="banner"></div>
            <ul class="nearby_shop_list">
                <li class="nearby_item" v-for="(item,index) in nearbyrList" :key="index" @click="openShop(item)">
                    <div class="item_top">
                        <div class="item_left">
                            <img :src="item.ShopLogo" alt="" class="item_left_img fade_in" lazy-load="true">
                            <div class="item_left_text">
                                <div class="name">
                                    <div class="name_left">
                                        <span class="rest" v-if="item.OpenState==0">休息中</span>
                                        <p class="shop_name">{{item.ShopName}}</p>
                                    </div>
                                    <span class="distance">{{item.Distance}}km</span>
                                </div>
                                <p class="address">{{item.ShopAddress}}</p>
                                <div class="option" v-if="item.PaotuiPriceOff"><i class="icon_set"></i><span class="coupon_text">{{item.PaotuiPriceOff}}</span></div>
                                <div class="option" v-if="item.Coupons"><i class="icon icon_coupon"></i><span class="coupon_text">{{item.Coupons}}</span></div>
                                <div class="option" v-if="!item.PaotuiPriceOff&&!item.Coupons"><span class="no_coupon_text">{{item.NoPerMessage}}</span></div>
                            </div>
                        </div>
                    </div>
                    <div class="li_mask" v-if="item.OpenState==0"></div>
                </li>
                <div class="no_more" v-if="nomore">附近没有更多店铺了</div>
            </ul>
            <!-- </scroll-view> -->
        </div>
    </div>
</template>

<script>
    import gcoord from 'gcoord';
    export default {
        data() {
            return {
                winWidth: 0,
                winHeight: 0,
                nearbyrList: [],
                mapInfo: {},
                loginInfo: {},
                page: 1,
                quest: true,
                nomore: false,
                block: false,
                noPos: false, //无地理位置
            }
        },
        onLoad() {
            this.block = false;
            wx.showLoading({
                title: '加载中',
                mask: true
            })
            this.noPos = false;
        },
        onReady() { //页面渲染就会触发
            this.page = 1;
            this.quest = true;
            this.nomore = false;
            this.nearbyrList = [];
            this.util.qqMapInfo().then(res => {
                console.log(res)
                this.mapInfo = wx.getStorageSync('QQmap')
                this.nearbyShop()
            }).catch(err => {
                console.log(err)
                this.noPos = true;
            })
        },
        onShow() {
            //搜索位置获取
            if (wx.getStorageSync('QQmap') && !wx.getStorageSync('QQmap').mapGet) {
                this.mapInfo = wx.getStorageSync('QQmap')
                this.page = 1;
                this.nomore = false;
                this.nearbyShop()
            }
        },
        onPullDownRefresh() { //下拉刷新
            if (wx.getStorageSync('QQmap')) {
                this.page = 1;
                this.nomore = false;
                this.mapInfo = wx.getStorageSync('QQmap')
                if (this.mapInfo.mapGet) {
                    this.util.getLoc().then(res => {
                        // console.log('下拉')
                        this.nearbyShop(this.page)
                    }).catch(err => {
                        // this.msg('位置信息获取错误，请稍后重试')
                    })
                } else {
                    this.nearbyShop(this.page)
                }
            } else {
                wx.stopPullDownRefresh();
                this.msg('位置获取失败')
            }
        },
        onReachBottom() { //触底事件
            if (wx.getStorageSync('QQmap')) {
                if (this.quest) {
                    this.page++;
                    this.nearbyShop()
                }
            }
        },
        methods: {
            scrollHandler() {
                if (this.quest) {
                    this.page++;
                    this.nearbyShop()
                }
            },
            //附近的店铺
            nearbyShop(pg) {
                this.util.post({
                        url: '/api/Customer/Browse/NearbyShops',
                        data: {
                            CityName: this.mapInfo.city,
                            UserLocalPoint: this.trans(this.mapInfo),
                            Distance: 5000,
                            PageSize: 10,
                            PageIndex: this.page
                        }
                    })
                    .then(res => {
                        this.block = true;
                        wx.hideLoading();
                        wx.stopPullDownRefresh()
                        res.Body.Shops.forEach(e => {
                            e.ShopAddress = e.ShopAddress.replace(/\n/g, '').split('($)').join('');
                            e.Distance = Math.floor(e.Distance / 1000 * 100) / 100;
                            e.ShopLogo = e.ShopLogo + '?x-oss-process=image/resize,w_100/format,jpg';
                        })
                        if (pg == 1) {
                            this.nearbyrList = res.Body.Shops;
                            this.quest = true;
                            this.nomore = false;
                        } else {
                            this.nearbyrList.push(...res.Body.Shops);
                        }
                        if (!res.Body.Shops.length && this.page == 1) {
                            this.msg('您的附近暂时还没有店铺哦')
                        } else if (!res.Body.Shops.length && this.page > 1) {
                            this.quest = false;
                            this.nomore = true;
                        } else if (res.Body.Shops.length > 1 && res.Body.Shops.length < 10 && this.page > 1) {
                            this.quest = false;
                            this.nomore = true;
                        }
                    }).catch(err => {
                        wx.hideLoading();
                        this.msg(err.Msg)
                    })
            },
            openShop(info) {
                let {
                    ShopId,
                    ShopTemplateId
                } = info;
                // console.log(ShopId, ShopTemplateId)
                if (ShopTemplateId == 1) {
                    wx.navigateTo({
                        url: `/pages/food-store/main?ShopId=${ShopId}&type=1`
                    })
                } else {
                    wx.navigateTo({
                        url: `/pages/business/main?ShopId=${ShopId}&type=1`
                    })
                }
            },
            search() {
                wx.navigateTo({
                    url: `/pages/search-shop/main`
                })
            },
            trans(pos) {
                let {
                    latitude,
                    longitude
                } = pos;
                var result = gcoord.transform(
                    [latitude, longitude], // 经纬度坐标
                    gcoord.WGS84, // 当前坐标系
                    gcoord.BD09, //  目标坐标系
                );
                let location = `${result[1]},${result[0]}`;
                return location;
            },
            searchAddress() {
                wx.navigateTo({
                    url: '/pages/select-address/main?type=2'
                })
            }
        },
        components: {}
    }
</script>

<style lang="less">
    .nearby_shop {
        background: #fff;
    }
    .nearby_top {
        padding: 0 36rpx 16rpx;
    }
    .search {
        height: 80rpx;
        display: flex;
        align-items: center;
        justify-content: flex-start;
        overflow: hidden;
        .text {
            margin-left: 12rpx;
            font-size: 28rpx;
            color: #999;
        }
    }
    .nearby_shop_list {
        overflow: hidden;
        .nearby_item {
            background: #fff;
            position: relative;
            .item_top {
                padding: 36rpx 0;
                margin: 0 36rpx;
                display: flex;
                align-items: center;
                overflow: hidden;
                position: relative;
                .item_left {
                    display: flex;
                    justify-content: flex-start; // align-items: center;
                    flex: 1;
                    .item_left_img {
                        width: 120rpx;
                        height: 120rpx;
                        display: block;
                        border-radius: 8rpx;
                    }
                    .item_left_text {
                        padding-left: 30rpx;
                        flex: 1;
                        max-width: 525rpx;
                        .name {
                            width: 100%;
                            display: flex;
                            justify-content: space-between;
                            transform: translateY(-2rpx);
                            .name_left {
                                flex: 1;
                                display: flex;
                                align-items: center;
                                max-width: 390rpx;
                                .rest {
                                    width: 72rpx;
                                    height: 30rpx;
                                    line-height: 30rpx;
                                    text-align: center;
                                    background-color: #595959;
                                    border-radius: 2rpx;
                                    font-size: 20rpx;
                                    color: #fff;
                                    white-space: nowrap;
                                    margin-right: 10rpx;
                                }
                                .shop_name {
                                    overflow: hidden;
                                    text-overflow: ellipsis;
                                    white-space: nowrap;
                                    font-size: 32rpx;
                                    color: #1a1a1a;
                                    font-weight: 700;
                                    width: 80%;
                                }
                            }
                        }
                        .address {
                            margin-top: 10rpx;
                            overflow: hidden;
                            text-overflow: ellipsis;
                            white-space: nowrap;
                            font-size: 26rpx;
                            color: #999;
                            width: 100%;
                        }
                        .option {
                            height: 40rpx;
                            display: flex;
                            justify-content: flex-start;
                            align-items: center;
                            margin-top: 10rpx;
                            p,
                            span {
                                color: #1a1a1a;
                                font-size: 26rpx;
                                white-space: nowrap;
                                text-overflow: ellipsis;
                                overflow: hidden;
                                line-height: 42rpx;
                                padding: 0 8rpx;
                                flex: 1;
                            }
                            .coupon_text {
                                padding-right: 20rpx;
                            }
                            .no_coupon_text {
                                padding: 0;
                            }
                        }
                    }
                    .distance {
                        font-size: 24rpx;
                        color: #b3b3b3;
                        font-weight: normal;
                    }
                }
                .item_right_img {
                    width: 60rpx;
                    height: 60rpx;
                    display: block;
                }
                &:after {
                    content: '';
                    display: block;
                    width: 100%;
                    height: 0;
                    border-bottom: 1px solid #ebebeb;
                    position: absolute;
                    left: 0;
                    bottom: 0;
                    transform: scaleY(0.5);
                    transform-origin: 0 0;
                    z-index: 10;
                }
            }
        }
        .li_mask {
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background: rgba(255, 255, 255, 0.6);
            z-index: 2;
        }
    }
    .no_more {
        height: 70rpx;
        width: 100%;
        line-height: 70rpx;
        font-size: 20rpx;
        color: #999;
        text-align: center;
        background: #f3f3f3;
    }
    .no_pos {
        position: absolute;
        width: 100%;
        height: 100%;
        display: flex;
        align-items: center;
        flex-direction: column;
        .no_pos_con {
            padding-top: 300rpx;
            display: flex;
            align-items: center;
            flex-direction: column;
            p {
                font-size: 24rpx;
                color: #999;
                margin: 30rpx 0 40rpx;
            }
            .search_address {
                width: 184rpx;
                height: 64rpx;
                background-color: #ff4d3a;
                border-radius: 6rpx;
                font-size: 26rpx;
                line-height: 64rpx;
                color: #fff;
                text-align: center;
            }
        }
    }
</style>
