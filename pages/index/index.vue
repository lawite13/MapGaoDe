<template>
  <view class="index-page">
    <view id="map-container" class="map-container"></view>
    
    <!-- Floating button container -->
    <view class="button-container">
      <view class="top-buttons">
        <button @click="undoAction" :disabled="!canUndo" class="icon-button">
          <i class="fas fa-undo"></i>
        </button>
        <button @click="redoAction" :disabled="!canRedo" class="icon-button">
          <i class="fas fa-redo"></i>
        </button>
      </view>
      <view class="bottom-buttons">
        <button @click="startDrawing" :class="{'active': isDrawing}">編輯</button>
        <button @click="finishDrawing" :class="{'active': !isDrawing}">移動地圖</button>
      </view>
      <view class="color-picker">
        <label for="line-color">顔色:</label>
        <input type="color" id="line-color" v-model="lineColor" @input="handleColorChange" />
      </view>
    </view>
  </view>
</template>

<script>
export default {
  name: "IndexPage",
  data() {
    return {
      map: null,
      isDrawing: false,
      canvas: null,
      ctx: null,
      linePath: [],
      history: [],  // Store the drawing history
      redoStack: [],  // Store the redo actions
      lineColor: "#00209E", // Default color for new drawings
      currentLineColor: "#00209E", // The color for the current line being drawn
      mapCenter: [114.1694, 22.3193],
      mapZoom: 12,
      mapMoving: false, // Flag to track if map is moving
    };
  },
  mounted() {
    this.initializeMap();
  },
  computed: {
    canUndo() {
      return this.history.length > 0;
    },
    canRedo() {
      return this.redoStack.length > 0;
    },
  },
  methods: {
    // Initialize the map
    initializeMap() {
      this.map = new AMap.Map("map-container", {
        zoom: this.mapZoom,
        center: this.mapCenter,
        mapStyle: "amap://styles/normal",
      });

      // Set up canvas for drawing
      const canvasContainer = document.createElement("canvas");
      const mapContainer = this.map.getContainer();
      mapContainer.appendChild(canvasContainer);
      this.canvas = canvasContainer;
      this.ctx = this.canvas.getContext("2d");

      // Position the canvas correctly on the map
      this.canvas.style.position = 'absolute';
      this.canvas.style.top = '0';
      this.canvas.style.left = '0';
      this.canvas.style.pointerEvents = 'none';

      this.resizeCanvas();
      window.addEventListener("resize", this.resizeCanvas);

      // Listen for zoom, move, and real-time updates
      this.map.on("zoomend", this.redrawLines);
      this.map.on("moveend", this.redrawLines);
      this.map.on("move", this.handleMapMove);  // Track map movement for opacity adjustment
      this.map.on("zoom", this.handleMapMove);  // Same for zooming
    },

    resizeCanvas() {
      const size = this.map.getSize();
      this.canvas.width = size.width;
      this.canvas.height = size.height;
      this.redrawLines();
    },

    // Start drawing
    startDrawing() {
      this.isDrawing = true;
      this.map.on("mousedown", this.handleMouseDown);
      this.map.on("mousemove", this.handleMouseMove);
      document.addEventListener("mouseup", this.handleMouseUp);

      // Disable map panning while drawing
      this.map.setStatus({ dragEnable: false });
    },

    // Stop drawing
    finishDrawing() {
      this.isDrawing = false;
      this.map.off("mousedown", this.handleMouseDown);
      this.map.off("mousemove", this.handleMouseMove);
      document.removeEventListener("mouseup", this.handleMouseUp);

      // Re-enable map panning
      this.map.setStatus({ dragEnable: true });
    },

    // Handle mouse down for line drawing
    handleMouseDown(e) {
      this.mouseDown = true;
      const { lng, lat } = e.lnglat;
      if (isNaN(lng) || isNaN(lat)) return;

      this.linePath = [[lng, lat]];
      this.ctx.beginPath();
      const pixel = this.map.lngLatToContainer(new AMap.LngLat(lng, lat));
      this.ctx.moveTo(pixel.x, pixel.y);
    },

    // Handle mouse move for line drawing
    handleMouseMove(e) {
      if (!this.mouseDown) return;

      const { lng, lat } = e.lnglat;
      if (isNaN(lng) || isNaN(lat)) return;

      this.linePath.push([lng, lat]);

      const pixel = this.map.lngLatToContainer(new AMap.LngLat(lng, lat));
      this.ctx.lineTo(pixel.x, pixel.y);
      this.ctx.strokeStyle = this.currentLineColor; // Apply current line color
      this.ctx.lineWidth = 6;
      this.ctx.lineCap = 'round';
      this.ctx.lineJoin = 'round';
      this.ctx.stroke();
    },

    // Handle mouse up for line drawing
    handleMouseUp() {
      if (!this.mouseDown) return;
      this.mouseDown = false;
      this.history.push({ path: [...this.linePath], color: this.currentLineColor });
      this.linePath = [];
      this.redoStack = [];
    },

    // Undo the last action
    undoAction() {
      if (!this.canUndo) return;
      const lastLine = this.history.pop();
      this.redoStack.push(lastLine);
      this.redrawLines();
    },

    // Redo the last undone action
    redoAction() {
      if (!this.canRedo) return;
      const lineToRedo = this.redoStack.pop();
      this.history.push(lineToRedo);
      this.redrawLines();
    },

    // Redraw all lines from history
    redrawLines() {
      // Clear the canvas before redrawing
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

      // Redraw all lines from history
      this.history.forEach((line) => {
        this.ctx.beginPath();

        line.path.forEach((point, index) => {
          const [lng, lat] = point;
          const pixelPosition = this.map.lngLatToContainer(new AMap.LngLat(lng, lat));

          if (index === 0) {
            this.ctx.moveTo(pixelPosition.x, pixelPosition.y);
          } else {
            this.ctx.lineTo(pixelPosition.x, pixelPosition.y);
          }
        });

        // Apply opacity based on map movement state
        const opacity = this.mapMoving ? 0.3 : 1;
        this.ctx.globalAlpha = opacity;

        this.ctx.strokeStyle = line.color; // Use the color stored for each line
        this.ctx.lineWidth = 6;
        this.ctx.lineCap = 'round';
        this.ctx.lineJoin = 'round';
        this.ctx.stroke();

        // Reset globalAlpha for next drawing
        this.ctx.globalAlpha = 1;
      });
    },

    // Handle map movement (track map move state for opacity adjustment)
    handleMapMove() {
      this.mapMoving = true;
      this.redrawLines(); // Redraw lines with reduced opacity while map is moving
    },

    // Handle map move end (restore opacity when map stops moving)
    handleMapStop() {
      this.mapMoving = false;
      this.redrawLines(); // Redraw lines with normal opacity
    },

    // Handle color change for new lines only
    handleColorChange() {
      this.currentLineColor = this.lineColor; // Set the new color for next drawing
    },
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.resizeCanvas);
    this.map.off("move", this.handleMapMove);
    this.map.off("zoom", this.handleMapMove);
  },
};
</script>

<style scoped>
.map-container {
  width: 100%;
  height: 80vh;
  position: relative;
}

.button-container {
  bottom: 20px;
  right: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
}

.top-buttons, .bottom-buttons {
  display: flex;
  gap: 10px;
}

button {
  background-color: #007bff;
  color: white;
  border: none;
  padding: 10px;
  border-radius: 5px;
  cursor: pointer;
}

button.active {
  background-color: #0056b3;
}

button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

button:hover:not(:disabled) {
  background-color: #0056b3;
}

.color-picker {
  position: absolute;
  bottom: 100px;
  right: 20px;
  background-color: white;
  padding: 10px;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.color-picker input {
  margin-left: 10px;
}
</style>
