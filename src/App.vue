<template>
  <div class="d-none">
    <!-- Upload Section - Animated to Top -->
    <div class="upload-container" :class="{ collapsed: isExpanded }">
      <div class="upload-section" :class="{ show: isExpanded }">
        <label for="fileInput" class="upload-label">
          <div class="upload-content">
            <div>
              <button class="btn btn-danger" @click="toggleCollapse">×</button>

              <svg class="upload-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path d="M12 4v12m0 0l-3-3m3 3l3-3M4 16v2a2 2 0 002 2h12a2 2 0 002-2v-2" stroke-width="2"
                  stroke-linecap="round" />
              </svg>
              <div class="upload-text">
                <span class="upload-title">Choose a file or drag & drop</span>
                <span class="upload-subtitle">Trello JSON Export</span>
              </div>
            </div>
          </div>
        </label>
      </div>
    </div>
  </div>


  <section class="dark-mode h-100 d-flex flex-column">
    <nav class="navbar navbar-dark bg-dark mb-3">
      <div class="container-fluid flex-nowrap overflow-auto">
        <div class="d-flex align-items-center me-auto">
          <label for="fileInput" class="btn btn-sm btn-outline-primary me-2 text-nowrap">📂 Upload</label>
          <input type="file" id="fileInput" @change="onFileChange" hidden>

          <button @click="downloadData" class="btn btn-sm btn-outline-success me-2 text-nowrap">💾 Download</button>
          <button @click="SaveLocalStorage" class="btn btn-sm btn-outline-info me-2 text-nowrap">📥 Save</button>

          <button @click="ClearLocalStorage" class="btn btn-sm btn-outline-danger me-2 text-nowrap">🗑️ Clear</button>

          <div class="form-check form-switch d-flex align-items-center ms-3 text-nowrap">
            <input type="checkbox" v-model="autoSaveCheckbox" class="form-check-input" id="autoSaveCheckbox" />
            <label class="form-check-label ms-2 text-light" for="autoSaveCheckbox">Auto Save</label>
          </div>


          <div class="form-check form-switch d-flex align-items-center ms-3 text-nowrap">
            <input type="checkbox" v-model="labelNameCheckbox" class="form-check-input" id="labelNameCheckbox" />
            <label class="form-check-label ms-2 text-light" for="labelNameCheckbox">Hide Label name</label>
          </div>

          <div class="form-check form-switch d-flex align-items-center ms-3 text-nowrap">
            <input type="checkbox" v-model="closedCheckbox" class="form-check-input" id="closedSwitch" />
            <label class="form-check-label ms-2 text-light" for="closedSwitch">Show Closed</label>
          </div>
        </div>

        <div class="d-flex align-items-center ms-3">
          <div class="btn-group">
            <button @click="zoomIn" class="btn btn-sm btn-outline-secondary text-nowrap">🔍+</button>
            <button @click="zoomOut" class="btn btn-sm btn-outline-secondary text-nowrap">🔍-</button>
            <button @click="resetZoom" class="btn btn-sm btn-outline-secondary text-nowrap">⟳</button>
          </div>
        </div>
      </div>
    </nav>

    <!-- Sidebar Toggle Button -->
    <button class="sidebar-toggle" @click="toggleSidebar" :class="{ active: sidebarOpen }">
      <span v-if="!sidebarOpen">☰</span>
      <span v-else>×</span>
    </button>

    <!-- Sidebar -->
    <div class="sidebar" :class="{ open: sidebarOpen }">
      <!-- Tab Navigation -->
      <div class="sidebar-tabs">
        <button v-for="tab in sidebarTabs" :key="tab.id" class="sidebar-tab" :class="{ active: activeTab === tab.id }"
          @click="activeTab = tab.id">
          {{ tab.label }}
        </button>
      </div>

      <!-- Tab Content -->
      <div class="sidebar-content">
        <!-- Main Tab -->
        <div v-if="activeTab === 'main' && this.$refs.viewerRef" class="tab-panel">
          <h5>Board Info</h5>
          <div class="sidebar-field">
            <label>Board Name:</label>
            <input v-model="this.$refs.viewerRef.main.name" type="text" class="form-control" />
          </div>
          <div class="sidebar-field" v-if="this.$refs.viewerRef.main.lists">
            <label>Lists: {{ this.$refs.viewerRef.main.lists.length ?? 0}}</label>
          </div>
          <div class="sidebar-field" v-if="this.$refs.viewerRef.main.cards">
            <label>Cards: {{ this.$refs.viewerRef.main.cards.length ?? 0 }}</label>
          </div>
          <div class="sidebar-field" v-if="this.$refs.viewerRef.main.checklists">
            <label>checklist: {{ this.$refs.viewerRef.main.checklists.length ?? 0}}</label>
          </div>
          <div class="sidebar-field" v-if="this.$refs.viewerRef.main.exiting_tags">
            <label>tags: {{ this.$refs.viewerRef.main.exiting_tags.length ?? 0}}</label>
          </div>
          <div class="sidebar-field" v-if="this.$refs.viewerRef.main.dateLastActivity">
            <label>dateLastActivity: {{ formatDate(this.$refs.viewerRef.main.dateLastActivity) }}</label>
          </div>
          <div class="sidebar-field" v-if="this.$refs.viewerRef.main.dateLastActivity">
            <label>dateLastView: {{ formatDate(this.$refs.viewerRef.main.dateLastActivity) }}</label>
          </div>

        </div>

        <!-- Data Tab -->
        <div v-if="activeTab === 'data'" class="tab-panel">
          <h5>Data Management</h5>
          <button @click="SaveLocalStorage" class="btn btn-sm btn-outline-info w-100 mb-2">📥 Save</button>
          <button @click="downloadData" class="btn btn-sm btn-outline-success w-100 mb-2">💾 Download</button>
          <button @click="ClearLocalStorage" class="btn btn-sm btn-outline-danger w-100">🗑️ Clear</button>
        </div>

        <!-- Label Tab -->
        <div v-if="activeTab === 'label' && this.$refs.viewerRef"" class=" tab-panel">
          <h5>Default Labels</h5>
          <div class="trello-sidebar-labels-list">
            <div  v-for="(value, key) in this.$refs.viewerRef.main.labelNames" :key="key"
              class="trello-sidebar-label-item">
              <div class="trello-sidebar-label-visual">
                <LabelBar :labelKey="key" :labelColorMap="labelColorMap" />
              </div>
              <div class="trello-sidebar-label-info">
              </div>
              <input v-model="this.$refs.viewerRef.main.labelNames[key]" type="text" class="trello-sidebar-label-input" :placeholder="'Set a Name'" />
            </div>
          </div>
          <div class="text-muted">No tags found</div>
        </div>
      </div>
    </div>

    <!-- Sidebar Overlay -->
    <div v-if="sidebarOpen" class="sidebar-overlay" @click="toggleSidebar"></div>

    <Viewer ref="viewerRef" :labelColorMap="labelColorMap" :export_data="fileText" :is_archived="closedCheckbox" :labelNameCheckbox="labelNameCheckbox" @update-expand="toggleCollapse" />
  </section>

</template>

<script>
import Viewer from './components/Viewer.vue';
import LabelBar from './components/LabelBar.vue';

export default {
  name: 'FileReaderDemo',
  components: { Viewer,LabelBar },
  data() {
    return {
      zoomLevel: 100,
      fileText: '',          // holds the decoded text
      fileBinary: null,      // holds raw ArrayBuffer if needed
      isExpanded: true,
      closedCheckbox: false,
      autoSaveCheckbox: true,
      labelNameCheckbox:true,

      sidebarOpen: false,
      activeTab: 'main',
      sidebarTabs: [
        { id: 'main', label: 'Main' },
        { id: 'data', label: 'Data' },
        { id: 'label', label: 'Label' }
      ],

      labelColorMap: {
        black: '#1a1a1a',
        black_dark: '#0d0d0d',
        black_light: '#333333',
        blue: '#0079bf',
        blue_dark: '#005a8c',
        blue_light: '#4da6d9',
        green: '#519839',
        green_dark: '#3d732b',
        green_light: '#7bc86c',
        lime: '#6a9f3c',
        lime_dark: '#4f7a2d',
        lime_light: '#8bbd5a',
        orange: '#d29034',
        orange_dark: '#a86e25',
        orange_light: '#e0af6a',
        pink: '#cd5a91',
        pink_dark: '#a84472',
        pink_light: '#e08db4',
        purple: '#89609e',
        purple_dark: '#6b487c',
        purple_light: '#b08cc2',
        red: '#b04632',
        red_dark: '#8c3627',
        red_light: '#d17b6a',
        sky: '#00aecc',
        sky_dark: '#008699',
        sky_light: '#4dcde0',
        yellow: '#e6c60d',
        yellow_dark: '#b39b0a',
        yellow_light: '#b4a547',
        Done: '#61bd4f'
      }
    };
  },
  mounted() {
    this.autoLoadOnStartup();
    // Load saved zoom from localStorage if exists
    const savedZoom = localStorage.getItem('pageZoom');
    if (savedZoom) {
      this.zoomLevel = parseFloat(savedZoom);
      this.applyZoom();
    }
  },
  computed(){
  
  },
  methods: {

    toggleSidebar() {
      this.sidebarOpen = !this.sidebarOpen;
    },

    formatDate(dateString) {
      if (!dateString) return '';
      return new Date(dateString).toLocaleString('en-US', {
        year: 'numeric',
        month: 'short',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      });
    },


    generateLabelSVG(key) {
      const color = this.labelColorMap[key] || '#888';
      const w = 44;
      const h = 44;
      const padding = 4;
      const rectW = w - (padding * 2);
      const rectH = h - (padding * 2);
      const rectX = padding;
      const rectY = padding;
      const radius = 4;

      const patternMap = {
        black: 'stripes',
        black_dark: 'diagonal',
        black_light: 'dots',
        blue: 'dots_stripe',
        blue_dark: 'block_dot',
        blue_light: 'stripes',
        green: 'diagonal',
        green_dark: 'dots',
        green_light: 'dots_stripe',
        lime: 'block_dot',
        lime_dark: 'stripes',
        lime_light: 'diagonal',
        orange: 'dots',
        orange_dark: 'dots_stripe',
        orange_light: 'block_dot',
        pink: 'stripes',
        pink_dark: 'diagonal',
        pink_light: 'dots',
        purple: 'dots_stripe',
        purple_dark: 'block_dot',
        purple_light: 'stripes',
        red: 'diagonal',
        red_dark: 'dots',
        red_light: 'dots_stripe',
        sky: 'block_dot',
        sky_dark: 'stripes',
        sky_light: 'diagonal',
        yellow: 'dots',
        yellow_dark: 'dots_stripe',
        yellow_light: 'block_dot',
        Done: 'stripes'
      };

      const pattern = patternMap[key] || 'stripes';
      const darkerColor = this.adjustColorBrightness(color, -50);
      const lighterColor = this.adjustColorBrightness(color, 20);

      const safeKey = key.replace(/[^a-zA-Z0-9]/g, '_');
      const patternId = `pat-${safeKey}`;
      const clipId = `clip-${safeKey}`;
      const gradientId = `grad-${safeKey}`;

      // Simple rounded rectangle path
      const rectPath = `M ${rectX + radius} ${rectY} h ${rectW - radius * 2} a ${radius} ${radius} 0 0 1 ${radius} ${radius} v ${rectH - radius * 2} a ${radius} ${radius} 0 0 1 -${radius} ${radius} h -${rectW - radius * 2} a ${radius} ${radius} 0 0 1 -${radius} -${radius} v -${rectH - radius * 2} a ${radius} ${radius} 0 0 1 ${radius} -${radius} z`;

      let patternSVG = '';

      switch (pattern) {
        case 'stripes':
          patternSVG = `<pattern id="${patternId}" patternUnits="userSpaceOnUse" width="44" height="6"><rect width="44" height="3" fill="${darkerColor}"/><rect y="3" width="44" height="3" fill="${lighterColor}"/></pattern>`;
          break;
        case 'diagonal':
          patternSVG = `<pattern id="${patternId}" patternUnits="userSpaceOnUse" width="10" height="10"><rect width="10" height="10" fill="${lighterColor}"/><line x1="0" y1="0" x2="0" y2="10" stroke="${darkerColor}" stroke-width="5"/><line x1="5" y1="0" x2="5" y2="10" stroke="${lighterColor}" stroke-width="5"/></pattern>`;
          break;
        case 'dots':
          patternSVG = `<pattern id="${patternId}" patternUnits="userSpaceOnUse" width="10" height="10"><rect width="10" height="10" fill="${lighterColor}"/><circle cx="5" cy="5" r="2.5" fill="${darkerColor}"/></pattern>`;
          break;
        case 'dots_stripe':
          patternSVG = `<pattern id="${patternId}" patternUnits="userSpaceOnUse" width="12" height="44"><rect width="12" height="44" fill="${lighterColor}"/><rect y="19" width="12" height="6" fill="${darkerColor}"/><circle cx="6" cy="6" r="2" fill="${darkerColor}"/><circle cx="6" cy="38" r="2" fill="${darkerColor}"/></pattern>`;
          break;
        case 'block_dot':
          patternSVG = `<pattern id="${patternId}" patternUnits="userSpaceOnUse" width="44" height="44"><rect width="44" height="44" fill="${lighterColor}"/><rect x="14" y="14" width="16" height="16" rx="2" fill="${darkerColor}"/><circle cx="22" cy="22" r="3" fill="#ffffff"/><circle cx="8" cy="8" r="1.5" fill="${darkerColor}"/><circle cx="36" cy="8" r="1.5" fill="${darkerColor}"/><circle cx="36" cy="36" r="1.5" fill="${darkerColor}"/><circle cx="8" cy="36" r="1.5" fill="${darkerColor}"/></pattern>`;
          break;
      }

      return `<defs>
              <linearGradient id="${gradientId}" x1="0%" y1="0%" x2="100%" y2="100%">
                <stop offset="0%" style="stop-color:${color};stop-opacity:1" />
                <stop offset="100%" style="stop-color:${this.adjustColorBrightness(color, -30)};stop-opacity:1" />
              </linearGradient>
              ${patternSVG}
              <clipPath id="${clipId}"><path d="${rectPath}"/></clipPath>
            </defs>
            <path d="${rectPath}" fill="url(#${gradientId})" stroke="${this.adjustColorBrightness(color, -40)}" stroke-width="0.5"/>
            <rect width="44" height="44" fill="url(#${patternId})" clip-path="url(#${clipId})" opacity="0.6"/>`;
    },

    adjustColorBrightness(hex, percent) {
      let r, g, b;
      if (hex.startsWith('#')) {
        const num = parseInt(hex.replace('#', ''), 16);
        r = (num >> 16) & 0xFF;
        g = (num >> 8) & 0xFF;
        b = num & 0xFF;
      } else if (hex.startsWith('rgb')) {
        const match = hex.match(/\d+/g);
        if (match) {
          r = parseInt(match[0]);
          g = parseInt(match[1]);
          b = parseInt(match[2]);
        } else {
          return hex;
        }
      } else {
        return hex;
      }

      const amt = Math.round(2.55 * percent);
      const R = Math.min(255, Math.max(0, r + amt));
      const G = Math.min(255, Math.max(0, g + amt));
      const B = Math.min(255, Math.max(0, b + amt));
      return `rgb(${R},${G},${B})`;
    },

    toggleCollapse() {
      this.isExpanded = !this.isExpanded
    },

    SaveLocalStorage() {
      const dataToSave = this.$refs.viewerRef.main;
      localStorage.setItem('trello-board-data', JSON.stringify(dataToSave));
      alert('Data saved to browser storage!');
    },

    // Load from localStorage
    LoadLocalStorage() {
      const savedData = localStorage.getItem('trello-board-data');
      if (savedData) {
        try {
          const parsedData = JSON.parse(savedData);
          this.$refs.viewerRef.main = parsedData;
          alert('Data loaded from browser storage!');
        } catch (e) {
          alert('Error loading saved data');
        }
      } else {
        alert('No saved data found in browser storage');
      }
    },

    // Clear localStorage
    ClearLocalStorage() {
      if (confirm('Are you sure you want to clear saved data?')) {
        localStorage.removeItem('trello-board-data');
        alert('Saved data cleared!');
      }
    },

    // Auto-load on startup
    autoLoadOnStartup() {
      const savedData = localStorage.getItem('trello-board-data');
      if (savedData) {
        try {
          const parsedData = JSON.parse(savedData);
          this.$refs.viewerRef.main = parsedData;
          console.log('Auto-loaded data from browser storage');
        } catch (e) {
          console.error('Error auto-loading data:', e);
        }
      }
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

  }

};
</script>
