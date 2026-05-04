
<template>
  <div class="">
    <!-- Collapse/Expand Button -->
    <button class="collapse-btn" @click="toggleCollapse">
      <span class="btn-icon" :class="{ rotated: !isExpanded }">▼</span>
    </button>
    <!-- Upload Section - Animated to Top -->
    <div class="upload-container" :class="{ collapsed: !isExpanded }">
      <div class="upload-section" :class="{ show: isExpanded }">
          <label for="fileInput" class="upload-label">

              <div class="upload-content">
                <div>
                  <svg class="upload-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                      <path d="M12 4v12m0 0l-3-3m3 3l3-3M4 16v2a2 2 0 002 2h12a2 2 0 002-2v-2" stroke-width="2" stroke-linecap="round"/>
                  </svg>
                  <div class="upload-text">
                      <span class="upload-title">Choose a file or drag & drop</span>
                      <span class="upload-subtitle">Trello JSON Export</span>
                  </div>
                  </div>
              </div>
          </label>
          <input type="file" id="fileInput" @change="onFileChange" hidden>
      </div>
    </div>
  </div>


  <section class="dark-mode h-100 d-flex flex-column">
    <div class="nav bg-primary"><input type="checkbox" v-model="closedCheckbox"/>
    <div class="zoom-controls">
  <button @click="zoomIn" class="zoom-btn">🔍 Zoom In</button>
  <button @click="zoomOut" class="zoom-btn">🔍 Zoom Out</button>
  <button @click="resetZoom" class="zoom-btn">⟳ Reset</button>

  <button @click="downloadData" class="btn">
  📥 save Data
</button>
</div></div>
    
    <Viewer ref="viewerRef" :export_data="fileText" :is_archived="closedCheckbox"  @update-expand="toggleCollapse"/>
  </section>
  
</template>

<script>
import Viewer from './components/Viewer.vue';

export default {
  name: 'FileReaderDemo',
  components: { Viewer },
  data() {
    return {
      zoomLevel:100,
      fileText: '',          // holds the decoded text
      fileBinary: null,      // holds raw ArrayBuffer if needed
      isExpanded:true,
      closedCheckbox:false,
    };
  },
  mounted() {
    // Load saved zoom from localStorage if exists
    const savedZoom = localStorage.getItem('pageZoom');
    if (savedZoom) {
      this.zoomLevel = parseFloat(savedZoom);
      this.applyZoom();
    }
  },
  methods: {
    toggleCollapse() {
        this.isExpanded = !this.isExpanded
    },
    downloadData() {
      // Create a copy of the main data
      const dataToDownload = this.$refs.viewerRef.main
      
      // Convert to JSON string with pretty formatting
      const jsonString = JSON.stringify(dataToDownload, null, 2);
      
      // Create blob and download link
      const blob = new Blob([jsonString], { type: 'application/json' });
      const url = URL.createObjectURL(blob);
      const link = document.createElement('a');
      
      // Set download filename with current timestamp
      const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
      link.download = `trello-data-${timestamp}.json`;
      
      link.href = url;
      link.click();
      
      // Clean up
      URL.revokeObjectURL(url);
      
      // Optional: Show success message
      this.$emit('data-downloaded', dataToDownload);
    },
    onFileChange(event) {
      const file = event.target.files[0];
      if (!file) return;
      const reader = new FileReader();
      // For plain text files
      reader.onload = e => {
        this.fileText = e.target.result; // string
      };
      reader.readAsText(file); 
    },


    /* Zooming */
    applyZoom() {
      const targetDiv = document.getElementById('zoom-canvas');
      if (targetDiv) {
        targetDiv.style.transform = `scale(${this.zoomLevel / 100})`;
        targetDiv.style.transformOrigin = 'top left';
        targetDiv.style.width = `${100 / (this.zoomLevel / 100)}%`;
      }
      
      // Save to localStorage
      localStorage.setItem('pageZoom', this.zoomLevel);
    },
    
    zoomIn() {
      if (this.zoomLevel < 200) { // Max 200%
        this.zoomLevel += 10;
        this.applyZoom();
      }
    },
    
    zoomOut() {
      if (this.zoomLevel > 50) { // Min 50%
        this.zoomLevel -= 10;
        this.applyZoom();
      }
    },
    
    resetZoom() {
      this.zoomLevel = 100;
      this.applyZoom();
    },

  },
};
</script>
