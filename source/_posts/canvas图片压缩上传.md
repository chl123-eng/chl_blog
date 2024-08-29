---
title: canvas图片压缩上传
date: 2024-08-28 11:05:04
tags:
categories:
  - 常用组件封装
cover: https://chlblog.oss-cn-guangzhou.aliyuncs.com/compress.jpg
---

### 前言

uni-app实现将图片压缩后上传到服务器，实现方式：新建canvas画布，设置尺寸来实现压缩，同时显示了图片格式


### 代码实现
```
//PicCompress.vue
<template>
	<view class="compress" v-if="canvasId">
		<canvas :canvas-id="canvasId" :style="{ width: canvasSize.width, height: canvasSize.height }"></canvas>
	</view>
</template>
 
<script>
export default {
	data() {
		return {
			pic: '',
			canvasSize: {
				width: 0,
				height: 0
			},
			canvasId: ''
		};
	},
	mounted() {
		// 创建 canvasId
		if (!uni || !uni._helang_compress_canvas) {
			uni._helang_compress_canvas = 1;
		} else {
			uni._helang_compress_canvas++;
		}
		this.canvasId = `compress-canvas${uni._helang_compress_canvas}`;
	},
	methods: {
		// 压缩
		compressFun(params) {
			return new Promise(async (resolve, reject) => {
				// 等待图片信息
				let info = await this.getImageInfo(params.src)
					.then((info) => info)
					.catch(() => null);
 
				if (!info) {
					reject('获取图片信息异常');
					return;
				}
 
				// 设置最大 & 最小 尺寸
				const maxSize = params.maxSize || 1080;
				const minSize = params.minSize || 640;
 
				// 当前图片尺寸
				let { width, height } = info;
 
				// 非 H5 平台进行最小尺寸校验
				// #ifndef H5
				if (width <= minSize && height <= minSize) {
					resolve(params.src);
					return;
				}
				// #endif
 
				// 最大尺寸计算
				//（图像的宽度和高度是否超过最大尺寸。如果其中任一维度超过最大尺寸，代码将对图像进行调整，以使其适应最大尺寸并保持其宽高比。）
				// 这样可以确保图像在调整大小后仍保持原始比例，并且不会超过指定的最大尺寸
 
				if (width > maxSize || height > maxSize) {
					if (width > height) {
						height = Math.floor(height / (width / maxSize));
						width = maxSize;
					} else {
						width = Math.floor(width / (height / maxSize));
						height = maxSize;
					}
				}
 
				// 设置画布尺寸
				this.$set(this, 'canvasSize', {
					width: `${width}px`,
					height: `${height}px`
				});
 
				// Vue.nextTick 回调在 App 有异常，则使用 setTimeout 等待DOM更新
				setTimeout(() => {
					// 创建 canvas 绘图上下文（指定 canvasId）。在自定义组件下，第二个参数传入组件实例this，以操作组件内 <canvas/> 组件
					// Tip: 需要指定 canvasId，该绘图上下文只作用于对应的 <canvas/>
					const ctx = uni.createCanvasContext(this.canvasId, this);
					// 清除画布上在该矩形区域内的内容。（x，y，宽，高）
					ctx.clearRect(0, 0, width, height);
					// 绘制图像到画布。（所要绘制的图片资源，x，y，宽，高）
					ctx.drawImage(info.path, 0, 0, width, height);
					// 将之前在绘图上下文中的描述（路径、变形、样式）画到 canvas 中。
					// 本次绘制是否接着上一次绘制，即reserve参数为false，则在本次调用drawCanvas绘制之前native层应先清空画布再继续绘制；若reserver参数为true，则保留当前画布上的内容，本次调用drawCanvas绘制的内容覆盖在上面，默认 false
					// 绘制完成后回调
					ctx.draw(false, () => {
						// 把当前画布指定区域的内容导出生成指定大小的图片，并返回文件路径。在自定义组件下，第二个参数传入自定义组件实例，以操作组件内 <canvas> 组件。
						uni.canvasToTempFilePath(
							{
								x: 0, //画布x轴起点（默认0）
								y: 0, //画布y轴起点（默认0）
								width: width, //画布宽度（默认为canvas宽度-x）
								height: height, //画布高度（默认为canvas高度-y
								destWidth: width, //图片宽度（默认为 width * 屏幕像素密度）
								destHeight: height, //输出图片高度（默认为 height * 屏幕像素密度）
								canvasId: this.canvasId, //画布标识，传入 <canvas/> 的 canvas-id（支付宝小程序是id、其他平台是canvas-id）
								fileType: params.fileType || 'png', //目标文件的类型，只支持 'jpg' 或 'png'。默认为 'png'
								quality: params.quality || 0.9, //图片的质量，取值范围为 (0, 1]，不在范围内时当作1.0处理
								success: (res) => {
									// 在H5平台下，tempFilePath 为 base64
									resolve(res.tempFilePath);
								},
								fail: (err) => {
									reject(null);
								}
							},
							this
						);
					});
				}, 300);
			});
		},
		// 获取图片信息
		getImageInfo(src) {
			return new Promise((resolve, reject) => {
				uni.getImageInfo({
					src,
					success: (info) => {
						resolve(info);
					},
					fail: (err) => {
						console.log(err, 'err===获取图片信息');
						reject(null);
					}
				});
			});
		},
		// 批量压缩
		async compress(params) {
			// 初始化状态变量
			let [index, done, fail] = [0, 0, 0];
			let paths = [];
 
			// 处理待压缩图片列表 
			let waitList = Array.isArray(params.src) ? params.src : [params.src];
 
			// 批量压缩方法
			let batch = async () => {
				while (index < waitList.length) {
					try {
						const path = await next();
						done++;
						paths.push(path);
						params.progress?.({ done, fail, count: waitList.length });
					} catch (error) {
						fail++;
						params.progress?.({ done, fail, count: waitList.length });
					}
					index++;
				}
			};
 
			// 单个图片压缩方法
			let next = () => {
				const currentSrc = waitList[index];
				return this.compressFun({
					src: currentSrc,
					maxSize: params.maxSize,
					fileType: params.fileType,
					quality: params.quality,
					minSize: params.minSize
				});
			};
			// 返回Promise并处理结果
			return new Promise((resolve, reject) => {
				try {
					batch()
						.then(() => {
							if (typeof params.src === 'string') {
								resolve(paths[0]);
							} else {
								resolve(paths);
							}
						})
						.catch((error) => {
							reject(error);
						});
				} catch (error) {
					reject(error);
				}
			});
		}
	}
};
</script>
 
<style lang="scss" scoped>
.compress {
	position: fixed;
	width: 12px;
	height: 12px;
	overflow: hidden;
	top: -99999px;
	left: 0;
}
</style>

```
//使用实例
```
<template>
    <view>
        <FileCompress ref="helangCompress" />
        <view class="imgItem" @click="uploadImg()">上传文件</view>
    </view>
</template> 
<script>
import FileCompress from './FileCompress.vue';
import { BASE_URL } from '../../config'
export default{
    components: {
        FileCompress
    },
    data(){
        return {

        }
    },
    methods:{
        uploadImg() {
            let _this = this
            uni.chooseImage({
                count: 1,
                sizeType: ['original', 'compressed'],
                success: function(res) {
                    let imgUrl = res.tempFilePaths[0]
                    let fileType = ''
                    //根据原始图片后缀生成响应的图片格式 (此处可自行修改)
                    fileType = _this.getImageFormatFromUrl(imgUrl)
                    if (fileType != 'png') {
                        fileType = 'jpg'
                    }
                    // 单张压缩 
                    _this.$refs.helangCompress.compressFun({
                        src: imgUrl, //图片地址
                        maxSize: 800, //压缩后的最大尺寸
                        fileType: fileType, //压缩后的图片格式
                        quality: 0.1, //压缩的质量比 
                        minSize: -1 //最小压缩尺寸，图片尺寸小于该值时不压缩，非H5平台有效。若需要忽略该设置，可设置为一个极小的值，比如负数。
                    }).then((res) => {
                        // 压缩成功回调
                        console.log('压缩的成功回调', res)
                        //上传文件
                        uni.uploadFile({
                            url:BASE_URL + '/index.php/Api/Index/upload_file',
                            filePath: res,
                            name: 'file',
                            formData: {
                                path: 'path'
                            },
                            success: (res2) => {
                                
                            },
                            fail:(err)=>{
                                
                            }
                        });

                    }).catch((err) => {
                        // 压缩失败回调  
                        console.log('压缩的失败回调', err)
                        
                    })
                },
                fail: function(err) {
                    console.log('上传失败', err)
                    
                }
            });
        },
        //获取图片后缀
        getImageFormatFromUrl(url) {
            // 使用正则表达式找到URL中最后一个"."后的所有字符，这通常是文件扩展名
            const match = url.match(/\.([^.]+)$/);
            // 返回匹配到的文件扩展名，如果没有匹配到则返回null
            return match ? match[1] : null;
        }
    }
}

</script>

```


//封装兼容的图片上传文件
```
import {
  OSSPolicy,
  OSSUpload,
  OSSPolicyAuthorized,
  OSSPolicyAuthorizedGet
} from '@/service/oss'
import { WBIAOID } from '~/constant'
const DIR = 'mall'
export default {
  data() {
    return {
      // 0 为普通照片上传，1为隐私照片上传
      uploadType: 0,
      isOpenCompress: false // 是否开启图片压缩
    }
  },
  props: {
    maxSize: {
      type: Number,
      default: 10
    }
  },
  methods: {
    // 上传之前
    beforeUpload(file) {
      if (file.size / 1024 / 1024 > this.maxSize) {
        this.$toast('您选择的图片太大了，请重新请选择')
        return false
      }
    },
    // 获取OSS - token
    async getOssToken() {
      const OSSArray = [OSSPolicy, OSSPolicyAuthorized]
      const data = await OSSArray[this.uploadType]({
        $axios: this.$axios,
        params: {
          dir: this.uploadType ? '' : DIR
        }
      })
      return data
    },
    async upload({ success, files, quality = 60 } = {}) {
      // 1. 上传前检查
      this.beforeUpload(files)
      // 2. 获取token
      const data = await this.getOssToken()
      const results = []
      // 3. 开始上传
      if (files.length) {
        for (let i = 0; i < files.length; i++) {
          const file = files[i]
          let bolb = file
          // 大于300KB才走压缩流程并且开启了压缩的字段
          if (file.size / 1024 > 300 && this.isOpenCompress) {
            const baseImg = await this.fileToBase64ByQuality(files[i], quality)
            bolb = this.convertBase64UrlToBlob(baseImg, file.type)
          }
          const params = new FormData()
          const aliyunFileKey =
            data.dir +
            new Date().getTime() +
            (this.$cookies.get(WBIAOID) || '') +
            file.name.replace(/[\u4e00-\u9fa5]/g, '')
          const uploadObj = {
            key: aliyunFileKey,
            OSSAccessKeyId: data.accessid,
            policy: data.policy,
            signature: data.signature,
            success_action_status: '200', // 让服务端返回200,不然，默认会返回204
            file: bolb // 文件必须声明在最后，否则异常
          }
          for (const i in uploadObj) {
            params.append(i, uploadObj[i])
          }
          try {
            const url = await OSSUpload({
              $axios: this.$axios,
              url: data.host.replace(/http/gi, 'https'),
              params
            })
            if (!url) {
              results.push(aliyunFileKey)
              success(results)
            }
          } catch (e) {
            this.$toast('上传失败')
          }
        }
      }
    },
    // 获取隐私图片方法
    async getAuthorized(dir) {
      const data = await OSSPolicyAuthorizedGet({
        $axios: this.$axios,
        params: {
          dir
        }
      })
      return data
    },
    //blob转换成base64:https://www.jb51.net/article/280974.htm
    fileToBase64ByQuality(file, quality) {
      const fileReader = new FileReader()
      const type = file.type
      return new Promise((resolve, reject) => {
        if (window.URL || window.webkitURL) {
          resolve(this.compress(URL.createObjectURL(file), quality, type))
        } else {
          fileReader.onload = () => {
            resolve(this.compress(fileReader.result, quality, type))
          }
          fileReader.onerror = e => {
            reject(e)
          }
          fileReader.readAsDataURL(file)
        }
      })
    },
    //  图片最大宽度

    /**
     * base64压缩（图片-canvas互转）
     * @param {file} base64 base64图片数据
     * @param {number} quality 图片质量
     * @param {string} format 输出图片格式
     * @return {base64} data 图片处理完成后的base64
     */
    compress(base64, quality, mimeType) {
      const MAX_WIDTH = 800
      const cvs = document.createElement('canvas')
      const img = document.createElement('img')
      img.crossOrigin = 'anonymous'
      return new Promise((resolve, reject) => {
        img.src = base64
        // 图片偏移值
        // let offetX = 0
        img.onload = () => {
          if (img.width > MAX_WIDTH) {
            cvs.width = MAX_WIDTH
            cvs.height = (img.height * MAX_WIDTH) / img.width
            // offetX = (img.width - MAX_WIDTH) / 2
          } else {
            cvs.width = img.width
            cvs.height = img.height
          }
          const ctx = cvs.getContext('2d')
          if (img.width > img.height) {
            ctx.translate(cvs.width / 2, cvs.height / 2) // 先位移坐标到中心点
            // ctx.rotate((90 * Math.PI) / 180) // 旋转90度
            // 此时按照旋转后的尺寸
            // 把定位中心移动到左上角
            ctx.translate(-cvs.width / 2, -cvs.height / 2)
            // 绘制图片
            ctx.drawImage(img, 0, 0, cvs.width, cvs.height)
            // 坐标系还原到初始
            ctx.setTransform(1, 0, 0, 1, 0, 0)
          } else {
            // 绘制图片
            ctx.drawImage(img, 0, 0, cvs.width, cvs.height)
          }
          const imageData = cvs.toDataURL(mimeType, quality / 100)
          resolve(imageData)
        }
      })
    },
    convertBase64UrlToBlob(base64, mimeType) {
      const bytes = window.atob(base64.split(',')[1])
      const ab = new ArrayBuffer(bytes.length)
      const ia = new Uint8Array(ab)
      for (let i = 0; i < bytes.length; i++) {
        ia[i] = bytes.charCodeAt(i)
      }
      const _blob = new Blob([ia], { type: mimeType })
      return _blob
    }
  }
}

```