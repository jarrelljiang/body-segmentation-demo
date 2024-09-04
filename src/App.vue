<script setup>
import * as bodySegmentation from "@tensorflow-models/body-segmentation";

import { onMounted, ref } from "vue";

let url = ref("");
onMounted(() => {
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

        // Segmenter can be null if initialization failed (for example when loading
        // from a URL that does not exist).
        if (segmenter != null) {
            // Detectors can throw errors, for example when using custom URLs that
            // contain a model that doesn't provide the expected output.
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
                        (singleSegmentation) => singleSegmentation.segmentation
                    );
                }
            } catch (error) {
                segmenter.dispose();
                segmenter = null;
                alert(error);
            }
        }
        if (segmentation && segmentation.length > 0) {
            // const data = await bodySegmentation.toBinaryMask(
            //     segmentation,
            //     { r: 0, g: 0, b: 0, a: 0 },
            //     { r: 0, g: 0, b: 0, a: 255 }, //表示遮罩的背景色，这里设为白色
            //     false,
            //     options.foregroundThreshold
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

            // await bodySegmentation.drawMask(
            //     canvas,
            //     video,
            //     data,
            //     options.maskOpacity,//遮罩背景色的透明度,默认配置的是0.7
            //     options.maskBlur
            // );
            url.value = myCanvas.toDataURL("image/png", 0);
        }
    }

    async function updateVideo(event) {
        // Clear reference to any previous uploaded video.
        URL.revokeObjectURL(video.currentSrc);
        const file = event.target.files[0];
        source.src = URL.createObjectURL(file);

        // Wait for video to be loaded.
        video.load();
        await new Promise((resolve) => {
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
    <div id="main">
        <div class="container">
            <div id="top-bar">
                <label for="videofile">Upload a video file:</label>
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
        </div>
    </div>
    <img :src="url" alt="" />
    <canvas
        id="myCanvas"
        width="320"
        height="180"
        style="visibility: hidden"
    ></canvas>
</template>

<style scoped>
body {
    margin: 0;
}
#main {
    margin: 0;
    position: relative;
}
#canvas-wrapper {
    margin-top: 80px;
    position: relative;
}
#top-bar {
    margin-left: 300px;
    position: relative;
}
</style>
