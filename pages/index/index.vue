<template>
  <view class="index-page">
    <view id="map-container" class="map-container"></view>

	<view class="icon-palette">
	  <img
		v-for="item in iconList"
		:key="item.type"
		:src="item.src"
		class="icon-thumb"
		draggable="true"
		@dragstart="onDragStart(item.type)"
	  />
	</view>
	<view class="icon-layer">
	  <img
		v-for="(icon, index) in placedIcons"
		:key="index"
		:src="getIconSrc(icon.type)"
		:style="getIconStyle(icon)"
		class="icon-on-map"
	  />
	</view>
	
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
        <button @click="startDrawing" :class="{ active: isDrawing }">編輯</button>
        <button @click="finishDrawing" :class="{ active: !isDrawing }">移動地圖</button>
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
      history: [],
      redoStack: [],
      lineColor: "#00209E",
      currentLineColor: "#00209E",
      mapCenter: [114.1694, 22.3193],
      mapZoom: 12,
      mapMoving: false,
      mouseDown: false,
	  draggingIconType: null,
	  placedIcons: [],
	  iconList: [
		{ type: "police", src: "/static/icons/police.png" },
		{ type: "barrier", src: "/static/icons/barrier.png" },
		{ type: "enemy", src: "/static/icons/enemy.png" },
		{ type: "target", src: "/static/icons/target.png" },
	  ],
    };
  },
  mounted() {
    this.initializeMap();
	this.canvas.addEventListener("dragover", (e) => e.preventDefault());
	this.canvas.addEventListener("drop", this.handleDrop);
	
	// Add touch event listeners for dragging icons
	this.canvas.addEventListener("touchstart", this.onTouchStart, { passive: false });
	this.canvas.addEventListener("touchmove", this.onTouchMove, { passive: false });
	this.canvas.addEventListener("touchend", this.onTouchEnd);
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
    initializeMap() {
      this.map = new AMap.Map("map-container", {
        zoom: this.mapZoom,
        center: this.mapCenter,
        mapStyle: "amap://styles/normal",
      });

      const canvasContainer = document.createElement("canvas");
      const mapContainer = this.map.getContainer();
      mapContainer.appendChild(canvasContainer);
      this.canvas = canvasContainer;
      this.ctx = this.canvas.getContext("2d");

      this.canvas.style.position = 'absolute';
      this.canvas.style.top = '0';
      this.canvas.style.left = '0';
      this.canvas.style.width = '100%';
      this.canvas.style.height = '100%';
      this.canvas.style.pointerEvents = 'none'; // Initially off

      this.resizeCanvas();
      window.addEventListener("resize", this.resizeCanvas);

      this.map.on("zoomend", this.redrawLines);
      this.map.on("moveend", () => {
        this.mapMoving = false;
        this.redrawLines();
      });
      this.map.on("move", this.handleMapMove);
      this.map.on("zoom", this.handleMapMove);
    },

    resizeCanvas() {
      const size = this.map.getSize();
      this.canvas.width = size.width;
      this.canvas.height = size.height;
      this.redrawLines();
    },

    startDrawing() {
      this.isDrawing = true;
      this.mouseDown = false; // Reset in case
    
      // Attach mouse events directly to the canvas
      this.canvas.addEventListener("mousedown", this.handleMouseDown);
      this.canvas.addEventListener("mousemove", this.handleMouseMove);
      document.addEventListener("mouseup", this.handleMouseUp);
    
      // Add touch events directly on canvas
      this.canvas.addEventListener("touchstart", this.handleTouchStart, { passive: false });
      this.canvas.addEventListener("touchmove", this.handleTouchMove, { passive: false });
      this.canvas.addEventListener("touchend", this.handleTouchEnd);
    
      this.canvas.style.pointerEvents = "auto"; // ✅ Enable interaction
      this.map.setStatus({ dragEnable: false });
    },


    finishDrawing() {
      this.isDrawing = false;
      
		// Remove event listeners for mouse and touch events
		this.canvas.removeEventListener("mousedown", this.handleMouseDown);
		this.canvas.removeEventListener("mousemove", this.handleMouseMove);
		document.removeEventListener("mouseup", this.handleMouseUp);
	  
		this.canvas.removeEventListener("touchstart", this.handleTouchStart);
		this.canvas.removeEventListener("touchmove", this.handleTouchMove);
		this.canvas.removeEventListener("touchend", this.handleTouchEnd);
	  
		this.canvas.style.pointerEvents = "none"; // ✅ Restore so map works
		this.map.setStatus({ dragEnable: true });
    },

    handleMouseDown(e) {
      // Get mouse position relative to the map container
      const rect = this.canvas.getBoundingClientRect();
      const x = e.clientX - rect.left; // Get the X coordinate of the mouse relative to the canvas
      const y = e.clientY - rect.top;  // Get the Y coordinate of the mouse relative to the canvas
    
      const lngLat = this.map.containerToLngLat([x, y]); // Convert to Lat/Lng using map's containerToLngLat method
      this.mouseDown = true;
      this.linePath = [[lngLat.lng, lngLat.lat]];
      this.ctx.beginPath();
      this.ctx.moveTo(x, y);
    },

    handleMouseMove(e) {
      if (!this.mouseDown) return;
    
      // Get the mouse position relative to the map container (canvas)
      const rect = this.canvas.getBoundingClientRect();
      const x = e.clientX - rect.left;
      const y = e.clientY - rect.top;
    
      // Convert the mouse position to Lat/Lng using map's containerToLngLat method
      const lngLat = this.map.containerToLngLat([x, y]);
    
      // Draw the line on canvas
      this.linePath.push([lngLat.lng, lngLat.lat]);
    
      this.ctx.lineTo(x, y);
      this.ctx.strokeStyle = this.currentLineColor;
      this.ctx.lineWidth = 6;
      this.ctx.lineCap = "round";
      this.ctx.lineJoin = "round";
      this.ctx.stroke();
    },
	
    handleMouseUp() {
      if (!this.mouseDown) return;
      this.mouseDown = false;
      this.history.push({ path: [...this.linePath], color: this.currentLineColor });
      this.linePath = [];
      this.redoStack = [];
    },
	// This will handle the start of the drag event for touch devices
	onTouchStart(e) {
	  const touch = e.touches[0];
	  const rect = this.canvas.getBoundingClientRect();
	  const x = touch.clientX - rect.left;
	  const y = touch.clientY - rect.top;
	
	  // Find the icon being dragged based on where the touch event happened
	  const icon = this.placedIcons.find(icon => {
	    const pos = this.map.lngLatToContainer(new AMap.LngLat(icon.lng, icon.lat));
	    return (
	      Math.abs(pos.x - x) < 30 &&
	      Math.abs(pos.y - y) < 30
	    );
	  });
	
	  if (icon) {
	    this.draggingIconType = icon.type;
	    this.draggingIcon = icon; // Track the icon being dragged
	  }
	},
	
	onTouchMove(e) {
	  const touch = e.touches[0];
	  const rect = this.canvas.getBoundingClientRect();
	  const x = touch.clientX - rect.left;
	  const y = touch.clientY - rect.top;
	
	  if (this.draggingIcon) {
	    const lngLat = this.map.containerToLngLat([x, y]);
	
	    // Update the icon's position based on the touch
	    this.draggingIcon.lng = lngLat.lng;
	    this.draggingIcon.lat = lngLat.lat;
	
	    // Redraw icons in the new position
	    this.redrawIcons();
	  }
	},
	
	// This will handle the end of the touch drag event
	onTouchEnd() {
	  if (this.draggingIcon) {
	    this.draggingIcon = null; // Stop dragging
	  }
	},
	
    handleTouchStart(e) {
      e.preventDefault();
      if (e.touches.length !== 1) return;

      const touch = e.touches[0];
      const rect = this.canvas.getBoundingClientRect();
      const x = touch.clientX - rect.left;
      const y = touch.clientY - rect.top;

      const lngLat = this.map.containerToLngLat([x, y]);
      this.mouseDown = true;
      this.linePath = [[lngLat.lng, lngLat.lat]];
      this.ctx.beginPath();
      this.ctx.moveTo(x, y);
    },

    handleTouchMove(e) {
      e.preventDefault();
      if (!this.mouseDown || e.touches.length !== 1) return;

      const touch = e.touches[0];
      const rect = this.canvas.getBoundingClientRect();
      const x = touch.clientX - rect.left;
      const y = touch.clientY - rect.top;

      const lngLat = this.map.containerToLngLat([x, y]);
      this.linePath.push([lngLat.lng, lngLat.lat]);

      this.ctx.lineTo(x, y);
      this.ctx.strokeStyle = this.currentLineColor;
      this.ctx.lineWidth = 6;
      this.ctx.lineCap = "round";
      this.ctx.lineJoin = "round";
      this.ctx.stroke();
    },

    handleTouchEnd() {
      if (!this.mouseDown) return;
      this.mouseDown = false;
      this.history.push({ path: [...this.linePath], color: this.currentLineColor });
      this.linePath = [];
      this.redoStack = [];
    },

    undoAction() {
      if (!this.canUndo) return;
      const lastLine = this.history.pop();
      this.redoStack.push(lastLine);
      this.redrawLines();
    },

    redoAction() {
      if (!this.canRedo) return;
      const lineToRedo = this.redoStack.pop();
      this.history.push(lineToRedo);
      this.redrawLines();
    },

    redrawLines() {
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

      this.history.forEach((line) => {
        this.ctx.beginPath();

        line.path.forEach((point, index) => {
          const [lng, lat] = point;
          const pixel = this.map.lngLatToContainer(new AMap.LngLat(lng, lat));
          if (index === 0) {
            this.ctx.moveTo(pixel.x, pixel.y);
          } else {
            this.ctx.lineTo(pixel.x, pixel.y);
          }
        });

        this.ctx.globalAlpha = this.mapMoving ? 0.3 : 1;
        this.ctx.strokeStyle = line.color;
        this.ctx.lineWidth = 6;
        this.ctx.lineCap = "round";
        this.ctx.lineJoin = "round";
        this.ctx.stroke();
        this.ctx.globalAlpha = 1;
      });
    },

    handleMapMove() {
      this.mapMoving = true;
      this.redrawLines(); // Redraw any lines being drawn on the map
    
      // Redraw icons based on their lat/lng coordinates
      this.redrawIcons();
    },

    handleColorChange() {
      this.currentLineColor = this.lineColor;
    },
	
	onDragStart(type) {
	    this.draggingIconType = type;
	},
	handleDrop(e) {
	  e.preventDefault();
	
	  const rect = this.canvas.getBoundingClientRect();
	  const x = e.clientX - rect.left;
	  const y = e.clientY - rect.top;
	
	  const lngLat = this.map.containerToLngLat([x, y]);
	
	  // Add the new icon with its geographical position
	  this.placedIcons.push({
	    type: this.draggingIconType,
	    lng: lngLat.lng,
	    lat: lngLat.lat,
	    posX: x, // Position on screen
	    posY: y, // Position on screen
	  });
	
	  this.draggingIconType = null; // Reset the dragging icon type
	},
	getIconSrc(type) {
	    const icon = this.iconList.find((i) => i.type === type);
	    return icon ? icon.src : "";
	},
	getIconStyle(icon) {
	  const pos = this.map.lngLatToContainer(new AMap.LngLat(icon.lng, icon.lat)); // Convert Lat/Lng to Pixel
	  
	  return {
	    left: `${pos.x - 16}px`, // Center the icon
	    top: `${pos.y - 16}px`, // Center the icon
	    width: '32px',
	    height: '32px',
	    position: 'absolute',
	  };
	},
	redrawIcons() {
		this.placedIcons.forEach(icon => {
	    // Recalculate the position of each icon on the map after it has moved
	    const pos = this.map.lngLatToContainer(new AMap.LngLat(icon.lng, icon.lat));
	    icon.posX = pos.x;
	    icon.posY = pos.y;
	  });
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

.top-buttons,
.bottom-buttons {
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
.icon-palette {
  position: absolute;
  bottom: 160px;
  left: 10px;
  display: flex;
  gap: 10px;
  background: white;
  padding: 10px;
  border-radius: 10px;
}

.icon-thumb {
  width: 40px;
  height: 40px;
  cursor: grab;
}

.icon-layer {
  position: absolute;
  top: 0;
  left: 0;
  pointer-events: none;
}

.icon-on-map {
  pointer-events: none;
}
</style>
