<template>
    <div>
        <input v-model.number="desiredWidth" type="number" placeholder="Width">
        <input v-model.number="desiredHeight" type="number" placeholder="Height">
        <div v-if="this.rect">
            Current size: {{ currentWidth | Math.round }} x {{ currentHeight | Math.round }}
        </div>
        <button @click="uploadImage">Upload Image</button>
        <div ref="container"></div>
    </div>
</template>

<script>
import Konva from 'konva';
import VueKonva from 'vue-konva';

export default {
    components: {
        VueKonva
    },
    data() {
        return {
            image: null,
            desiredWidth: 100,
            desiredHeight: 100,
            currentWidth: 100,
            currentHeight: 100,
            stage: null,
            layer: null,
            rect: null,
            transformer: null
        };
    },
    mounted() {
        this.stage = new Konva.Stage({
            container: this.$refs.container,
            width: window.innerWidth,
            height: window.innerHeight
        });
        this.layer = new Konva.Layer();
        this.stage.add(this.layer);
    },
    watch: {
        desiredWidth(newVal) {
            if (this.rect) {
                this.updateRectSize(newVal, null);
            }
        },
        desiredHeight(newVal) {
            if (this.rect) {
                this.updateRectSize(null, newVal);
            }
        }
    },
    methods: {
        uploadImage() {
            let input = document.createElement('input');
            input.type = 'file';
            input.onchange = e => {
                let file = e.target.files[0];
                let reader = new FileReader();
                reader.onload = e => {
                    let img = new Image();
                    img.onload = () => {
                        this.setupCanvas(img);
                    };
                    img.src = e.target.result;
                };
                reader.readAsDataURL(file);
            };
            input.click();
        },
        setupCanvas(img) {
            this.layer.destroyChildren(); // 清除所有子元素
            this.image = img;

            // 计算缩放比例以适应画板
            const scaleX = this.stage.width() / img.width;
            const scaleY = this.stage.height() / img.height;
            const scale = Math.min(scaleX, scaleY, 1); // 不放大图片

            let konvaImage = new Konva.Image({
                image: img,
                x: 0,
                y: 0,
                width: img.width * scale,
                height: img.height * scale
            });
            this.layer.add(konvaImage);
            this.addCropRect();
        },
        addCropRect() {
            if (this.rect) {
                this.rect.destroy();
                this.transformer.destroy();
            }

            this.rect = new Konva.Rect({
                x: 50,
                y: 50,
                width: this.desiredWidth,
                height: this.desiredHeight,
                fill: 'rgba(0,0,255,0.5)',
                draggable: true
            });
            this.layer.add(this.rect);

            // 添加移动时的边界检查
            this.rect.on('dragmove', () => {
                const rectX = this.rect.x();
                const rectY = this.rect.y();
                const rectWidth = this.rect.width();
                const rectHeight = this.rect.height();
                const imageWidth = this.image.width * this.image.scaleX();
                const imageHeight = this.image.height * this.image.scaleY();

                const newX = Math.min(Math.max(0, rectX), imageWidth - rectWidth);
                const newY = Math.min(Math.max(0, rectY), imageHeight - rectHeight);

                this.rect.x(newX);
                this.rect.y(newY);
            });

            this.rect.on('transformend', () => {
                this.currentWidth = this.rect.width() * this.rect.scaleX();
                this.currentHeight = this.rect.height() * this.rect.scaleY();
                this.layer.draw();
            });

            this.transformer = new Konva.Transformer({
                keepRatio: true,
                rotateEnabled: false,
                enabledAnchors: ['top-left', 'top-right', 'bottom-left', 'bottom-right']
            });
            this.layer.add(this.transformer);
            this.transformer.attachTo(this.rect);
            this.layer.draw();
        },
        updateRectSize(newWidth, newHeight) {
            const ratio = this.desiredWidth / this.desiredHeight;
            let width = newWidth || this.rect.width();
            let height = newHeight || this.rect.height();

            if (newWidth != null) {
                height = width / ratio;
            } else if (newHeight != null) {
                width = height * ratio;
            }

            // Adjust dimensions based on image boundaries
            if (height > this.image.height) {
                height = this.image.height;
                width = height * ratio;
            }
            if (width > this.image.width) {
                width = this.image.width;
                height = width / ratio;
            }

            this.rect.width(width);
            this.rect.height(height);
            this.transformer.forceUpdate();
            this.layer.draw();
        }
    }
};
</script>

<style scoped></style>