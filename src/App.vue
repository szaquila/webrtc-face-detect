<template>
	<div class="webrtc_face_detector">
		<div class="option">
			<div>
				<label>面板操作：</label>
				<button @click="fnOpen">启动摄像头视频媒体</button>
				<button @click="fnClose">结束摄像头视频媒体</button>
			</div>
			<div>
				<span style="margin-right: 20px">前置后置切换（重新启动摄像头）：</span>
				<label>
					前置
					<input type="radio" v-model="state.constraints.video.facingMode" value="user" />
				</label>
				<label>
					后置
					<input type="radio" v-model="state.constraints.video.facingMode" value="environment" />
				</label>
			</div>
			<div>
				<span style="margin-right: 20px">检测识别类型：</span>
				<label>
					轮廓检测
					<input type="radio" v-model="state.detection" value="landmark" />
				</label>
				<label>
					表情检测
					<input type="radio" v-model="state.detection" value="expression" />
				</label>
				<label>
					年龄性别检测
					<input type="radio" v-model="state.detection" value="age_gender" />
				</label>
			</div>
			<div>
				<label>
					边框或面部轮廓：
					<input type="checkbox" v-model="state.withBoxes" />
				</label>
				<label>检测人脸：</label>
				<label>
					可信单
					<input type="radio" v-model="state.detectFace" value="detectSingleFace" />
				</label>
				<label>
					模糊多
					<input type="radio" v-model="state.detectFace" value="detectAllFaces" />
				</label>
			</div>
			<div>
				<label>选择算法模型</label>
				<label>
					ssdMobilenetv1
					<input type="radio" v-model="state.nets" value="ssdMobilenetv1" />
				</label>
				<label>
					tinyFaceDetector
					<input type="radio" v-model="state.nets" value="tinyFaceDetector" />
				</label>
				<label>
					mtcnn
					<input type="radio" v-model="state.nets" value="mtcnn" />
				</label>
			</div>
		</div>
		<div class="see">
			<video id="myVideo" poster="https://dummyimage.com/640x480" muted loop playsinline @loadedmetadata="fnRun"></video>
			<canvas id="myCanvas" />
		</div>
	</div>
</template>

<script setup lang="ts">
import { reactive, nextTick, watch, onMounted, onBeforeUnmount } from 'vue'
import * as faceapi from 'face-api.js'

const state: any = reactive({
	nets: 'tinyFaceDetector',
	options: null,
	withBoxes: true, // 框或轮廓
	detectFace: 'detectSingleFace', // 单or多人脸
	detection: 'landmark',
	videoEl: null,
	canvasEl: null,
	timeout: 0,
	// 视频媒体参数配置
	constraints: {
		audio: false,
		video: {
			// ideal（应用最理想的）
			width: {
				min: 320,
				ideal: 640,
				max: 1920
			},
			height: {
				min: 240,
				ideal: 480,
				max: 1080
			},
			// frameRate受限带宽传输时，低帧率可能更适宜
			frameRate: {
				min: 15,
				ideal: 30,
				max: 60
			},
			// 显示模式前置后置
			facingMode: 'user'
		}
	}
})

onMounted(() => {
	nextTick(() => {
		fnInit()
	})
})
onBeforeUnmount(() => {
	fnClose()
})

// 初始化模型加载
const fnInit = async () => {
	// await faceapi.nets[state.nets].loadFromUri('/models') // 算法模型
	// 根据模型参数识别调整结果
	switch (state.nets) {
		case 'ssdMobilenetv1':
			await faceapi.loadSsdMobilenetv1Model('/models')
			state.options = new faceapi.SsdMobilenetv1Options({
				minConfidence: 0.5 // 0.1 ~ 0.9
			})
			break
		case 'tinyFaceDetector':
			await faceapi.loadTinyFaceDetectorModel('/models')
			state.options = new faceapi.TinyFaceDetectorOptions({
				inputSize: 512, // 160 224 320 416 512 608
				scoreThreshold: 0.5 // 0.1 ~ 0.9
			})
			break
		case 'mtcnn':
			await faceapi.loadMtcnnModel('/models')
			state.options = new faceapi.MtcnnOptions({
				minFaceSize: 20, // 0.1 ~ 0.9
				scaleFactor: 0.709 // 0.1 ~ 0.9
			})
			break
	}
	await faceapi.loadFaceLandmarkModel('/models') // 轮廓模型
	await faceapi.loadFaceExpressionModel('/models') // 表情模型
	await faceapi.loadAgeGenderModel('/models') // 年龄模型
	// 节点属性化
	state.videoEl = document.getElementById('myVideo')
	state.canvasEl = document.getElementById('myCanvas')
}
// 人脸面部勘探轮廓识别绘制
const fnRunFaceLandmark = async () => {
	// console.log('RunFaceLandmark')
	if (state.videoEl.paused) return clearTimeout(state.timeout)
	// 识别绘制人脸信息
	// const result = await faceapi[state.detectFace](state.videoEl, state.options).withFaceLandmarks()
	let result: any
	if (state.detectFace === 'detectSingleFace') {
		result = await faceapi.detectSingleFace(state.videoEl, state.options).withFaceLandmarks()
	} else {
		result = await faceapi.detectAllFaces(state.videoEl, state.options).withFaceLandmarks()
	}
	if (result && !state.videoEl.paused) {
		const dims = faceapi.matchDimensions(state.canvasEl, state.videoEl, true)
		const resizeResult = faceapi.resizeResults(result, dims)
		// state.withBoxes ? faceapi.draw.drawDetections(state.canvasEl, resizeResult) : faceapi.draw.drawFaceLandmarks(state.canvasEl, resizeResult)
		if (state.withBoxes) {
			faceapi.draw.drawDetections(state.canvasEl, resizeResult)
		} else {
			faceapi.draw.drawFaceLandmarks(state.canvasEl, resizeResult)
		}
	} else {
		state.canvasEl.getContext('2d').clearRect(0, 0, state.canvasEl.width, state.canvasEl.height)
	}
	state.timeout = setTimeout(() => fnRunFaceLandmark())
}
// 人脸表情识别绘制
const fnRunFaceExpression = async () => {
	// console.log('RunFaceExpression')
	if (state.videoEl.paused) return clearTimeout(state.timeout)
	// 识别绘制人脸信息
	// const result = await faceapi[state.detectFace](state.videoEl, state.options).withFaceLandmarks().withFaceExpressions()
	let result: any
	if (state.detectFace === 'detectSingleFace') {
		result = await faceapi.detectSingleFace(state.videoEl, state.options).withFaceLandmarks().withFaceExpressions()
	} else {
		result = await faceapi.detectAllFaces(state.videoEl, state.options).withFaceLandmarks().withFaceExpressions()
	}
	if (result && !state.videoEl.paused) {
		const dims = faceapi.matchDimensions(state.canvasEl, state.videoEl, true)
		const resizeResult = faceapi.resizeResults(result, dims)
		// state.withBoxes ? faceapi.draw.drawDetections(state.canvasEl, resizeResult) : faceapi.draw.drawFaceLandmarks(state.canvasEl, resizeResult)
		if (state.withBoxes) {
			faceapi.draw.drawDetections(state.canvasEl, resizeResult)
		} else {
			faceapi.draw.drawFaceLandmarks(state.canvasEl, resizeResult)
		}
		faceapi.draw.drawFaceExpressions(state.canvasEl, resizeResult, 0.05)
	} else {
		state.canvasEl.getContext('2d').clearRect(0, 0, state.canvasEl.width, state.canvasEl.height)
	}
	state.timeout = setTimeout(() => fnRunFaceExpression())
}
// 年龄性别识别绘制
const fnRunFaceAgeAndGender = async () => {
	// console.log('RunFaceAgeAndGender')
	if (state.videoEl.paused) return clearTimeout(state.timeout)
	// 识别绘制人脸信息
	// const result = await faceapi[state.detectFace](state.videoEl, state.options).withFaceLandmarks().withAgeAndGender()
	let result: any
	if (state.detectFace === 'detectSingleFace') {
		result = await faceapi.detectSingleFace(state.videoEl, state.options).withAgeAndGender()
	} else {
		result = await faceapi.detectAllFaces(state.videoEl, state.options).withAgeAndGender()
	}
	if (result && !state.videoEl.paused) {
		const dims = faceapi.matchDimensions(state.canvasEl, state.videoEl, true)
		const resizeResults = faceapi.resizeResults(result, dims)
		// state.withBoxes ? faceapi.draw.drawDetections(state.canvasEl, resizeResults) : faceapi.draw.drawFaceLandmarks(state.canvasEl, resizeResults)
		if (state.withBoxes) {
			faceapi.draw.drawDetections(state.canvasEl, resizeResults)
		} else {
			faceapi.draw.drawFaceLandmarks(state.canvasEl, resizeResults)
		}
		if (Array.isArray(resizeResults)) {
			resizeResults.forEach(result => {
				const { age, gender, genderProbability } = result
				let sex: string = '男'
				if (gender === 'female') {
					sex = '女'
				}
				new faceapi.draw.DrawTextField([`${Math.round(age)} 岁`, `${sex} (${Math.round(genderProbability)})`], result.detection.box.bottomLeft).draw(state.canvasEl)
			})
		} else {
			const { age, gender, genderProbability } = resizeResults
			let sex: string = '男'
			if (gender === 'female') {
				sex = '女'
			}
			new faceapi.draw.DrawTextField([`${Math.round(age)} 岁`, `${sex} (${Math.round(genderProbability)})`], resizeResults.detection.box.bottomLeft).draw(state.canvasEl)
		}
	} else {
		state.canvasEl.getContext('2d').clearRect(0, 0, state.canvasEl.width, state.canvasEl.height)
	}
	state.timeout = setTimeout(() => fnRunFaceAgeAndGender())
}
// 执行识别类型
const fnRun = async () => {
	if (state.detection === 'landmark') {
		fnRunFaceLandmark()
		return
	}
	if (state.detection === 'expression') {
		fnRunFaceExpression()
		return
	}
	if (state.detection === 'age_gender') {
		fnRunFaceAgeAndGender()
		return
	}
}
// 启动摄像头视频媒体
const fnOpen = () => {
	if (typeof window.stream === 'object') return
	clearTimeout(state.timeout)
	state.timeout = setTimeout(() => {
		clearTimeout(state.timeout)
		navigator.mediaDevices.getUserMedia(state.constraints).then(fnSuccess).catch(fnError)
	}, 300)
}
// 成功启动视频媒体流
const fnSuccess = (stream: any) => {
	window.stream = stream // 使流对浏览器控制台可用
	state.videoEl.srcObject = stream
	state.videoEl.play()
}
// 失败启动视频媒体流
const fnError = (error: any) => {
	console.log(error)
	alert('视频媒体流获取错误' + error)
}
// 结束摄像头视频媒体
const fnClose = () => {
	state.canvasEl.getContext('2d').clearRect(0, 0, state.canvasEl.width, state.canvasEl.height)
	state.videoEl.pause()
	clearTimeout(state.timeout)
	if (typeof window.stream === 'object') {
		window.stream.getTracks().forEach((track: any) => track.stop())
		window.stream = ''
		state.videoEl.srcObject = null
	}
}

watch(
	() => [state.nets, state.detection],
	(newVal, oldVal) => {
		if (newVal[0] == oldVal[0]) {
			fnInit()
		}
		if (newVal[1] == oldVal[1]) {
			state.videoEl.pause()
			setTimeout(() => {
				state.videoEl.play()
				setTimeout(() => fnRun(), 300)
			})
		}
	}
)
</script>

<style scoped>
html {
	height: 100%;
}
body {
	margin: 0;
	padding: 0;
	height: 100%;
	display: block;
}
button {
	height: 30px;
	border: 2px #42b983 solid;
	border-radius: 4px;
	background: #42b983;
	color: white;
	margin: 10px;
}
.see {
	position: relative;
}
.see canvas {
	position: absolute;
	top: 0;
	left: 0;
}
.option {
	padding-bottom: 20px;
}
.option div {
	padding: 10px;
	border-bottom: 2px #42b983 solid;
}
.option div label {
	margin-right: 20px;
}
</style>
