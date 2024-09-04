<script setup>
import * as bodySegmentation from "@tensorflow-models/body-segmentation";

import { onMounted, ref } from "vue";

let url = ref("");
let showDanmu = false;
let timer = null;
let danmaku;
let danmu;
const changeDanmu = () => {
    showDanmu = !showDanmu;
    if (showDanmu) {
        timer = setInterval(() => {
            let n = parseInt(Math.random() * 10);
            for (let i = 0; i < n; i++) {
                danmu.draw();
            }
        }, 500);
    } else {
        clearInterval(timer);
        let danmuItems = document.querySelectorAll(".danmaku span");
        danmuItems.forEach(item => {
            item.remove();
        });
    }
};
class Danmu {
    constructor() {
        this.width = danmaku.offsetWidth;
        this.height = danmaku.offsetHeight;
    }
    draw() {
        let span = document.createElement("span");
        span.innerText = `弹幕${parseInt(Math.random() * 1000)}`;
        let speed = Math.random() * (5 - 3) + 3;
        span.style.transition = `${speed}s linear`;
        danmaku.appendChild(span);
        span.style.left = this.width + "px";
        span.style.top = Math.random() * (this.height - 20) + "px";
        let spanWidth = span.offsetWidth;
        setTimeout(() => {
            span.style.left = -spanWidth + "px";
        });
        setTimeout(() => {
            span.remove();
        }, speed * 1000);
    }
}
onMounted(() => {
    danmaku = document.getElementById("danmaku");
    danmu = new Danmu();
    let segmenter;
    const video = document.getElementById("video");
    const source = document.getElementById("currentVID");
    const myCanvas = document.getElementById("myCanvas");
    const myCtx = myCanvas.getContext("2d");
    async function createSegmenter() {
        return bodySegmentation.createSegmenter("MediaPipeSelfieSegmentation", {
            runtime: "mediapipe",
            modelType: "general",
            solutionPath:
                "https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation@0.1.1675465747",
        });
    }

    async function renderResult() {
        let segmentation = null;
        if (segmenter != null) {
            try {
                if (segmenter.segmentPeople != null) {
                    segmentation = await segmenter.segmentPeople(video, {
                        flipHorizontal: false,
                        multiSegmentation: false,
                        segmentBodyParts: true,
                        segmentationThreshold: 0.5,
                    });
                } else {
                    segmentation = await segmenter.estimatePoses(video, {
                        flipHorizontal: false,
                    });
                    segmentation = segmentation.map(
                        singleSegmentation => singleSegmentation.segmentation
                    );
                }
            } catch (error) {
                segmenter.dispose();
                segmenter = null;
                alert(error);
            }
        }
        if (segmentation && segmentation.length > 0) {
            //这个方法获取到的就是人体区域透明其他区域黑色的图片数据，如果图片不完整，需要把canvas的宽高设为视频的宽高
            // const data = await bodySegmentation.toBinaryMask(
            //     segmentation,
            //     { r: 0, g: 0, b: 0, a: 0 }, //表示人体区域的颜色，这里设为透明
            //     { r: 0, g: 0, b: 0, a: 255 }, //表示遮罩的背景色，这里设为黑色
            //     false,
            //     0.5
            // );
            // 将 ImageData 绘制到 Canvas 上
            // myCtx.putImageData(data, 0, 0);

            myCtx.clearRect(0, 0, myCanvas.width, myCanvas.height);
            myCtx.drawImage(
                // 经验证 即使出现多人，也只有一个 segmentation
                await segmentation[0].mask.toCanvasImageSource(),
                0,
                0,
                myCanvas.width,
                myCanvas.height
            );
            myCtx.globalCompositeOperation = "source-out";
            myCtx.fillRect(0, 0, myCanvas.width, myCanvas.height);

            //这个方法是把视频画面与toBinaryMask生成的遮罩融合在一起
            // await bodySegmentation.drawMask(
            //     myCanvas,
            //     video,
            //     data,
            //     options.maskOpacity,//遮罩背景色的透明度,默认配置的是0.7
            //     options.maskBlur
            // );
            url.value = myCanvas.toDataURL("image/png", 0); //第二个参数0表示画质最差，节约性能，因为遮罩图片不需要高质量
            danmaku.style.maskImage = `url(${url.value})`;
        }
    }

    async function updateVideo(event) {
        // Clear reference to any previous uploaded video.
        URL.revokeObjectURL(video.currentSrc);
        const file = event.target.files[0];
        source.src = URL.createObjectURL(file);

        video.load();
        await new Promise(resolve => {
            video.onloadeddata = () => {
                resolve(video);
            };
        });
    }

    async function runFrame() {
        await renderResult();
        requestAnimationFrame(runFrame);
    }

    async function run() {
        video.play();
        await runFrame();
    }

    async function app() {
        segmenter = await createSegmenter();
        const runButton = document.getElementById("submit");
        runButton.onclick = run;

        const uploadButton = document.getElementById("videofile");
        uploadButton.onchange = updateVideo;
    }

    app();
});
</script>

<template>
    <input type="checkbox" @change="changeDanmu()" />开启弹幕
    <div id="main">
        <div class="container">
            <div id="top-bar">
                <input
                    type="file"
                    id="videofile"
                    name="video"
                    accept="video/*"
                />
                <button id="submit">Run</button>
            </div>
        </div>
        <div class="container" id="canvas-wrapper">
            <video id="video">
                <source id="currentVID" src="" type="video/mp4" />
            </video>
            <div class="danmaku" id="danmaku"></div>
        </div>
    </div>
    <!-- <img :src="url" alt="" /> -->
    <!-- 把canvas画布宽高设小一些是为了提高myCanvas.toDataURL生成图片的性能，生成小尺寸图片配合mask-size:cover也能达到效果 -->
    <canvas
        id="myCanvas"
        width="320"
        height="180"
        style="visibility: hidden"
    ></canvas>
</template>

<style scoped lang="less">
body {
    margin: 0;
}
#main {
    margin: 0;
    position: relative;
}
#canvas-wrapper {
    margin: 80px auto 0;
    position: relative;
    width: 640px;
    height: 360px;
}
#top-bar {
    margin-left: 300px;
    position: relative;
}
.danmaku {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 90%;
    mask-repeat: no-repeat;
    mask-size: cover;
    z-index: 2;
    overflow: hidden;
    :deep(span) {
        position: absolute;
        font-size: 20px;
        color: #fff;
        font-weight: bold;
        white-space: nowrap;
    }
}
</style>
