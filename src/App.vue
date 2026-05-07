<template>
  <section class="h-100 d-flex flex-column">

    <!-- Navigation Bar -->
    <nav class="navbar navbar-dark bg-dark">
      <div class="container-fluid flex-nowrap overflow-auto">

        <div class="d-flex align-items-center me-auto">
          <!-- Simple Board Selector without Bootstrap JS -->
          <div class="board-selector-wrapper me-2">

            <!-- Custom Dropdown Menu -->
            <div v-if="showBoardMenu && boards.length > 0" class="custom-dropdown">
              <div v-for="board in boards" :key="board.id" class="custom-dropdown-item"
                :class="{ active: board.id === activeBoardId }" @click="switchBoard(board.id); showBoardMenu = false">
                <div class="dropdown-item-content">
                  <span class="dropdown-item-name">{{ board.name }}</span>
                  <small class="dropdown-item-date">{{ formatDate(board.lastModified) }}</small>
                </div>
                <button @click.stop="deleteBoard(board.id)" class="dropdown-item-delete"
                  title="Delete Board">🗑️</button>
              </div>
              <div class="custom-dropdown-divider"></div>
              <div class="custom-dropdown-item new-board-item" @click="openBoardOverlay; showBoardMenu = false">
                ➕ Create New Board
              </div>
            </div>
          </div>

          <button @click="openBoardOverlay" class="btn btn-sm btn-outline-info me-2 text-nowrap">
            📋 {{ boards.length > 0 ? 'Boards' : 'Create Board' }}
          </button>

          <div class="vr me-2"></div>

          <label for="fileInput" class="btn btn-sm btn-outline-primary me-2 text-nowrap">📂 Upload</label>
          <input type="file" id="fileInput" @change="onFileChange" hidden accept=".json">

          <button @click="downloadData" class="btn btn-sm btn-outline-success me-2 text-nowrap"
            :disabled="!activeBoardId">💾 Download</button>
          <button @click="SaveLocalStorage" class="btn btn-sm btn-outline-info me-2 text-nowrap"
            :disabled="!activeBoardId">📥 Save</button>
          <button @click="resetEverything" class="btn btn-sm btn-outline-warning me-2 text-nowrap">
            🔄 Reset All
          </button>

          <div class="vr me-2"></div>

          <div class="form-check form-switch d-flex align-items-center ms-2 text-nowrap">
            <input type="checkbox" v-model="autoSaveCheckbox" class="form-check-input" id="autoSaveCheckbox" />
            <label class="form-check-label ms-2 text-light" for="autoSaveCheckbox" style="font-size: 0.85rem;">
              Auto Save
            </label>
          </div>
          <div class="form-check form-switch d-flex align-items-center ms-2 text-nowrap">
            <input type="checkbox" v-model="labelNameCheckbox" class="form-check-input" id="labelNameCheckbox" />
            <label class="form-check-label ms-2 text-light" for="labelNameCheckbox" style="font-size: 0.85rem;">
              Hide Labels
            </label>
          </div>
          <div class="form-check form-switch d-flex align-items-center ms-2 text-nowrap">
            <input type="checkbox" v-model="closedCheckbox" class="form-check-input" id="closedSwitch" />
            <label class="form-check-label ms-2 text-light" for="closedSwitch" style="font-size: 0.85rem;">
              Archived
            </label>
          </div>
        </div>

        <div class="d-flex align-items-center ms-3">
          <div class="btn-group">
            <button @click="zoomIn" class="btn btn-sm btn-outline-secondary text-nowrap" title="Zoom In">🔍+</button>
            <button @click="zoomOut" class="btn btn-sm btn-outline-secondary text-nowrap" title="Zoom Out">🔍-</button>
            <button @click="resetZoom" class="btn btn-sm btn-outline-secondary text-nowrap" title="Reset Zoom">
              {{ zoomLevel }}%
            </button>
          </div>
        </div>

      </div>
    </nav>


    <!-- Board Overlay Modal -->
<div v-if="showBoardOverlay" class="modal-overlay">
  <div class="modal-content board-modal">
    <div class="modal-header">
      <h3 class="modal-title">📋 Board Management</h3>
    
    </div>
    
    <div class="modal-body">
      <!-- Create New Board Section -->
      <div class="create-board-section mb-4">
        <h4>Create New Board</h4>
        <div class="input-group">
          <input v-model="newBoardName" type="text" placeholder="Enter board name..." class="form-control"
            @keyup.enter="createNewBoard" ref="newBoardInput" />
          <button @click="createNewBoard" class="btn btn-primary" :disabled="!newBoardName.trim()">
            ➕ Create
          </button>
        </div>
      </div>

      <!-- Import Board Section -->
      <div class="import-board-section mb-4">
        <h4>Import Board from File</h4>
        <div class="input-group">
          <input type="file" @change="importBoardFromFile" class="form-control" accept=".json"
            ref="importFileInput" />
        </div>
      </div>

      <!-- Existing Boards List -->
      <div v-if="boards.length > 0" class="existing-boards-section">
        <h4>Your Boards ({{ boards.length }})</h4>
        <div class="boards-list">
          <div v-for="board in boards" :key="board.id" class="board-card"
            @click="switchBoard(board.id); closeBoardOverlay()" :class="{ active: board.id === activeBoardId }">
            <div class="board-card-info">
              <div class="board-card-name">{{ board.name }}</div>
              <div class="board-card-meta">
                <small>Created: {{ formatDate(board.createdAt) }}</small>
                <small>Modified: {{ formatDate(board.lastModified) }}</small>
              </div>
            </div>
            <div class="board-card-actions">
              <button @click.stop="switchBoard(board.id); closeBoardOverlay()" class="btn btn-sm btn-outline-light"
                title="Open Board">
                📂
              </button>
              <button @click.stop="deleteBoard(board.id)" class="btn btn-sm btn-outline-danger"
                title="Delete Board">
                🗑️
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- No Boards Message -->
      <div v-else class="no-boards-message">
        <div class="text-center p-4">
          <div class="fs-1 mb-3">🚀</div>
          <h5>No Boards Yet</h5>
          <p class="text-muted">Create a new board or import one from a file to get started!</p>
        </div>
      </div>
    </div>
  </div>
</div>

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
        <div v-if="activeTab === 'main' && $refs.viewerRef && $refs.viewerRef.main" class="tab-panel">
          <h5>Board Info</h5>
          <div class="sidebar-field">
            <label>Board Name:</label>
            <input v-model="$refs.viewerRef.main.name" type="text" class="form-control" @change="onDataChanged" />
          </div>
          <div class="sidebar-field" v-if="$refs.viewerRef.main.lists">
            <label>Lists:</label>
            <span class="badge bg-primary">{{ $refs.viewerRef.main.lists.length ?? 0 }}</span>
          </div>
          <div class="sidebar-field" v-if="$refs.viewerRef.main.cards">
            <label>Cards:</label>
            <span class="badge bg-success">{{ $refs.viewerRef.main.cards.length ?? 0 }}</span>
          </div>
          <div class="sidebar-field" v-if="$refs.viewerRef.main.checklists">
            <label>Checklists:</label>
            <span class="badge bg-info">{{ $refs.viewerRef.main.checklists.length ?? 0 }}</span>
          </div>
          <div class="sidebar-field" v-if="$refs.viewerRef.main.exiting_tags">
            <label>Tags:</label>
            <span class="badge bg-warning">{{ $refs.viewerRef.main.exiting_tags.length ?? 0 }}</span>
          </div>
          <div class="sidebar-field" v-if="$refs.viewerRef.main.dateLastActivity">
            <label>Last Activity:</label>
            <span>{{ formatDate($refs.viewerRef.main.dateLastActivity) }}</span>
          </div>
        </div>

        <!-- Data Tab -->
        <div v-if="activeTab === 'data'" class="tab-panel">
          <h5>Data Management</h5>
          <button @click="SaveLocalStorage" class="btn btn-sm btn-outline-info w-100 mb-2" :disabled="!activeBoardId">📥
            Save Current Board</button>
          <button @click="downloadData" class="btn btn-sm btn-outline-success w-100 mb-2" :disabled="!activeBoardId">💾
            Download as JSON</button>
          <button @click="openBoardOverlay" class="btn btn-sm btn-outline-primary w-100 mb-2">
            📋 Manage Boards
          </button>
          <hr>
          <button @click="resetEverything" class="btn btn-sm btn-outline-danger w-100">
            🔄 Reset Everything
          </button>
        </div>

        <!-- Label Tab -->
        <div v-if="activeTab === 'label' && $refs.viewerRef && $refs.viewerRef.main" class="tab-panel">
          <h5>Default Labels</h5>
          <div v-if="$refs.viewerRef.main.labelNames" class="trello-sidebar-labels-list">
            <div v-for="(value, key) in $refs.viewerRef.main.labelNames" :key="key" class="trello-sidebar-label-item">
              <div class="trello-sidebar-label-visual">
                <LabelBar :labelKey="key" :labelColorMap="labelColorMap" />
              </div>
              <div class="trello-sidebar-label-info">
                <input v-model="$refs.viewerRef.main.labelNames[key]" type="text"
                  class="trello-sidebar-label-input form-control form-control-sm" :placeholder="'Set a Name'"
                  @change="onDataChanged" />
              </div>
            </div>
          </div>
          <div v-else class="text-muted">No labels found</div>
        </div>
      </div>
    </div>

    <!-- Sidebar Overlay -->
    <div v-if="sidebarOpen" class="sidebar-overlay" @click="toggleSidebar"></div>

    <!-- Main Viewer Component -->
    <Viewer ref="viewerRef" :labelColorMap="labelColorMap" :export_data="fileText" :is_archived="closedCheckbox" :labelNameCheckbox="labelNameCheckbox"/>

  </section>
</template>

<script>
import Viewer from './components/Viewer.vue';
import LabelBar from './components/LabelBar.vue';

export default {
  name: 'FileReaderDemo',
  components: { Viewer, LabelBar },
  data() {
    return {
      fileText: '',
      fileBinary: null,

      closedCheckbox: false,
      autoSaveCheckbox: true,
      labelNameCheckbox: true,

      isExpanded: true,

      zoomLevel: 100,

      sidebarOpen: false,
      activeTab: 'main',
      sidebarTabs: [
        { id: 'main', label: 'Main' },
        { id: 'data', label: 'Data' },
        { id: 'label', label: 'Label' }
      ],

      // Board Management
      boards: [],
      activeBoardId: null,
      showBoardOverlay: true, // Default to open on startup
      showBoardMenu: false,  // Custom dropdown state
      newBoardName: '',
      autoSaveInterval: null,

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
  computed: {
    activeBoard() {
      return this.boards.find(b => b.id === this.activeBoardId) || null;
    }
  },
  watch: {
    autoSaveCheckbox(newVal) {
      if (newVal) {
        this.startAutoSave();
      } else {
        this.stopAutoSave();
      }
    },
    closedCheckbox() {
      localStorage.setItem('showArchived', this.closedCheckbox);
    },
    labelNameCheckbox() {
      localStorage.setItem('hideLabels', this.labelNameCheckbox);
    }
  },
  mounted() {
    this.loadPreferences();
    this.loadBoards();

    // Close dropdown when clicking outside
    document.addEventListener('click', this.closeBoardMenuOnClickOutside);

    // Show board overlay on startup if no boards exist
    if (this.boards.length === 0) {
      this.showBoardOverlay = true;
    } else {
      // Load last active board
      const lastActiveBoardId = localStorage.getItem('lastActiveBoardId');
      if (lastActiveBoardId && this.boards.find(b => b.id === lastActiveBoardId)) {
        this.loadBoard(lastActiveBoardId);
      } else {
        this.loadBoard(this.boards[0].id);
      }
    }

    if (this.autoSaveCheckbox) {
      this.startAutoSave();
    }
  },
  beforeUnmount() {
    this.stopAutoSave();
    this.saveCurrentBoardData();
    document.removeEventListener('click', this.closeBoardMenuOnClickOutside);
  },
  methods: {
    // ===== BOARD MENU =====

    toggleBoardMenu() {
      this.showBoardMenu = !this.showBoardMenu;
    },

    closeBoardMenuOnClickOutside(event) {
      if (this.showBoardMenu) {
        const boardSelector = event.target.closest('.board-selector-wrapper');
        if (!boardSelector) {
          this.showBoardMenu = false;
        }
      }
    },

    // ===== PREFERENCES =====

    loadPreferences() {
      const savedZoom = localStorage.getItem('pageZoom');
      if (savedZoom) {
        this.zoomLevel = parseFloat(savedZoom);
        this.$nextTick(() => this.applyZoom());
      }

      const showArchived = localStorage.getItem('showArchived');
      if (showArchived !== null) {
        this.closedCheckbox = showArchived === 'true';
      }

      const hideLabels = localStorage.getItem('hideLabels');
      if (hideLabels !== null) {
        this.labelNameCheckbox = hideLabels === 'true';
      }
    },

    // ===== BOARD MANAGEMENT =====

    openBoardOverlay() {
      this.showBoardOverlay = true;
      this.showBoardMenu = false;
      this.newBoardName = '';
      this.$nextTick(() => {
        if (this.$refs.newBoardInput) {
          this.$refs.newBoardInput.focus();
        }
      });
    },

    closeBoardOverlay() {
      if (this.boards.length > 0 || !this.newBoardName.trim()) {
        this.showBoardOverlay = false;
      }
    },

    loadBoards() {
      const savedBoards = localStorage.getItem('trelloBoards');
      if (savedBoards) {
        try {
          this.boards = JSON.parse(savedBoards);
        } catch (e) {
          console.error('Error loading boards:', e);
          this.boards = [];
        }
      }
    },

    saveBoards() {
      localStorage.setItem('trelloBoards', JSON.stringify(this.boards));
    },

    createNewBoard() {
      const name = this.newBoardName.trim();
      if (!name) return;

      const newBoard = {
        id: this.generateBoardId(),
        name: name,
        createdAt: new Date().toISOString(),
        lastModified: new Date().toISOString()
      };

      this.boards.push(newBoard);
      this.saveBoards();

      // Switch to the new board
      this.switchBoard(newBoard.id);

      // Reset input
      this.newBoardName = '';

      // Focus back on input for quick creation of multiple boards
      this.$nextTick(() => {
        if (this.$refs.newBoardInput) {
          this.$refs.newBoardInput.focus();
        }
      });
    },

    importBoardFromFile(event) {
      const file = event.target.files[0];
      if (!file) return;

      const reader = new FileReader();
      reader.onload = (e) => {
        try {
          const boardData = JSON.parse(e.target.result);

          const boardName = boardData.name || file.name.replace('.json', '');

          const newBoard = {
            id: this.generateBoardId(),
            name: boardName || 'Imported Board',
            createdAt: new Date().toISOString(),
            lastModified: new Date().toISOString()
          };

          // Check storage before saving
          const dataString = JSON.stringify(boardData);
          if (dataString.length > 4 * 1024 * 1024) { // 4MB warning
            alert('Warning: This board is very large (over 4MB). It may cause storage issues.');
          }

          this.boards.push(newBoard);
          this.saveBoards();

          // Save the imported data
          try {
            localStorage.setItem(`board_${newBoard.id}`, dataString);
          } catch (storageError) {
            // If storage fails, remove the board and show error
            this.boards = this.boards.filter(b => b.id !== newBoard.id);
            this.saveBoards();
            alert('Error: Not enough storage space. Try clearing some old boards first.');
            return;
          }

          // Switch to the new board
          this.switchBoard(newBoard.id);

          // Reset file input
          if (this.$refs.importFileInput) {
            this.$refs.importFileInput.value = '';
          }
        } catch (error) {
          alert('Error importing board: Invalid JSON file');
          console.error('Import error:', error);
        }
      };
      reader.readAsText(file);
    },

    loadBoard(boardId) {
      const boardData = localStorage.getItem(`board_${boardId}`);
      if (boardData) {
        try {
          this.fileText = boardData;
          this.activeBoardId = boardId;
          localStorage.setItem('lastActiveBoardId', boardId);

          const board = this.boards.find(b => b.id === boardId);
          if (board) {
            board.lastViewed = new Date().toISOString();
            this.saveBoards();
          }
        } catch (e) {
          console.error('Error loading board:', e);
        }
      } else {
        // Initialize empty board data
        const emptyData = this.getEmptyBoardData();
        const dataString = JSON.stringify(emptyData);
        localStorage.setItem(`board_${boardId}`, dataString);
        this.fileText = dataString;
        this.activeBoardId = boardId;
        localStorage.setItem('lastActiveBoardId', boardId);
      }
    },

    switchBoard(boardId) {
      if (boardId === this.activeBoardId) return;

      // Save current board first
      this.saveCurrentBoardData();

      // Load new board
      this.loadBoard(boardId);
    },

    saveCurrentBoardData() {
      if (this.activeBoardId && this.$refs.viewerRef && this.$refs.viewerRef.main) {
        try {
          const currentData = this.$refs.viewerRef.main;
          const dataString = JSON.stringify(currentData);
          localStorage.setItem(`board_${this.activeBoardId}`, dataString);

          const board = this.boards.find(b => b.id === this.activeBoardId);
          if (board) {
            board.lastModified = new Date().toISOString();
            this.saveBoards();
          }
        } catch (error) {
          console.error('Error saving board:', error);
          // Silently fail - data might be too large
        }
      }
    },

    deleteBoard(boardId) {
      const board = this.boards.find(b => b.id === boardId);
      if (!board) return;

      if (confirm(`Are you sure you want to delete "${board.name}"?\n\nThis action cannot be undone!`)) {
        localStorage.removeItem(`board_${boardId}`);

        this.boards = this.boards.filter(b => b.id !== boardId);
        this.saveBoards();

        if (this.activeBoardId === boardId) {
          if (this.boards.length > 0) {
            this.loadBoard(this.boards[0].id);
          } else {
            this.activeBoardId = null;
            this.fileText = '';
            this.showBoardOverlay = true;
          }
        }
      }
    },

    resetEverything() {
      if (confirm('⚠️  WARNING: This will delete ALL boards and ALL data!\n\nThis action cannot be undone. Are you sure?')) {
        this.stopAutoSave();

        const keysToRemove = [];
        for (let i = 0; i < localStorage.length; i++) {
          const key = localStorage.key(i);
          if (key && (key.startsWith('board_') || key === 'trelloBoards' || key === 'lastActiveBoardId')) {
            keysToRemove.push(key);
          }
        }
        keysToRemove.forEach(key => localStorage.removeItem(key));

        this.boards = [];
        this.activeBoardId = null;
        this.fileText = '';

        this.showBoardOverlay = true;
        this.newBoardName = '';

        if (this.autoSaveCheckbox) {
          this.startAutoSave();
        }
      }
    },

    getEmptyBoardData() {
      return {
        name: 'New Board',
        lists: [],
        cards: [],
        checklists: [],
        exiting_tags: [],
        labelNames: {
          black: '', black_dark: '', black_light: '',
          blue: '', blue_dark: '', blue_light: '',
          green: '', green_dark: '', green_light: '',
          lime: '', lime_dark: '', lime_light: '',
          orange: '', orange_dark: '', orange_light: '',
          pink: '', pink_dark: '', pink_light: '',
          purple: '', purple_dark: '', purple_light: '',
          red: '', red_dark: '', red_light: '',
          sky: '', sky_dark: '', sky_light: '',
          yellow: '', yellow_dark: '', yellow_light: '',
          Done: ''
        },
        dateLastActivity: new Date().toISOString()
      };
    },

    generateBoardId() {
      let newId;
      do {
        const timestamp = Date.now().toString(36);
        const random = Math.random().toString(36).substring(2, 8);
        newId = 'board_' + timestamp + random;
      } while (this.boards.some(b => b.id === newId));
      return newId;
    },

    onDataChanged() {
      if (this.autoSaveCheckbox) {
        this.saveCurrentBoardData();
      }
    },

    // ===== AUTO SAVE =====

    startAutoSave() {
      this.stopAutoSave();
      this.autoSaveInterval = setInterval(() => {
        this.saveCurrentBoardData();
      }, 5000);
    },

    stopAutoSave() {
      if (this.autoSaveInterval) {
        clearInterval(this.autoSaveInterval);
        this.autoSaveInterval = null;
      }
    },

    // ===== EXISTING METHODS =====

    toggleSidebar() {
      this.sidebarOpen = !this.sidebarOpen;
    },

    toggleCollapse() {
      this.isExpanded = !this.isExpanded;
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

    SaveLocalStorage() {
      this.saveCurrentBoardData();
      alert('✅ Board data saved successfully!');
    },

    ClearLocalStorage() {
      if (this.activeBoardId) {
        if (confirm('Clear current board data? This board will be reset to empty.')) {
          const emptyData = this.getEmptyBoardData();
          const dataString = JSON.stringify(emptyData);
          localStorage.setItem(`board_${this.activeBoardId}`, dataString);
          this.fileText = dataString;
          alert('🗑️ Current board cleared!');
        }
      }
    },

    downloadData() {
      if (!this.$refs.viewerRef || !this.$refs.viewerRef.main) {
        alert('No data to download');
        return;
      }

      const dataToDownload = this.$refs.viewerRef.main;
      const jsonString = JSON.stringify(dataToDownload, null, 2);
      const blob = new Blob([jsonString], { type: 'application/json' });
      const url = URL.createObjectURL(blob);
      const link = document.createElement('a');

      const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
      const boardName = this.activeBoard?.name || 'board';
      link.download = `${boardName.replace(/\s+/g, '_')}-${timestamp}.json`;

      link.href = url;
      link.click();
      URL.revokeObjectURL(url);
    },

    onFileChange(event) {
      const file = event.target.files[0];
      if (!file) return;
      const reader = new FileReader();
      reader.onload = (e) => {
        try {
          JSON.parse(e.target.result);
          this.fileText = e.target.result;
          if (this.activeBoardId) {
            this.saveCurrentBoardData();
          }
        } catch (error) {
          alert('Invalid JSON file!');
        }
      };
      reader.readAsText(file);
    },

    // ===== Zooming =====
    applyZoom() {
      const targetDiv = document.getElementById('zoom-canvas');
      if (targetDiv) {
        targetDiv.style.transform = `scale(${this.zoomLevel / 100})`;
        targetDiv.style.transformOrigin = 'top left';
        targetDiv.style.width = `${100 / (this.zoomLevel / 100)}%`;
      }
      localStorage.setItem('pageZoom', this.zoomLevel.toString());
    },

    zoomIn() {
      if (this.zoomLevel < 200) {
        this.zoomLevel += 10;
        this.applyZoom();
      }
    },

    zoomOut() {
      if (this.zoomLevel > 50) {
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

<style scoped>
/* Custom Dropdown Styles */
.board-selector-wrapper {
  position: relative;
  display: inline-block;
}

.custom-dropdown {
  position: absolute;
  top: 100%;
  left: 0;
  margin-top: 4px;
  background: #1a1a2e;
  border: 1px solid #2a2a4e;
  border-radius: 8px;
  min-width: 300px;
  max-height: 400px;
  overflow-y: auto;
  z-index: 2000;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.5);
}

.custom-dropdown-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  cursor: pointer;
  transition: background 0.2s;
  color: #ccc;
}

.custom-dropdown-item:hover {
  background: #2a2a4e;
}

.custom-dropdown-item.active {
  background: #0d6efd;
  color: white;
}

.dropdown-item-content {
  flex: 1;
}

.dropdown-item-name {
  display: block;
  font-weight: 500;
}

.dropdown-item-date {
  font-size: 0.75rem;
  color: #888;
}

.custom-dropdown-item.active .dropdown-item-date {
  color: #ccc;
}

.dropdown-item-delete {
  background: none;
  border: none;
  color: #ff6b6b;
  cursor: pointer;
  font-size: 1.2rem;
  padding: 4px 8px;
  border-radius: 4px;
  transition: background 0.2s;
}

.dropdown-item-delete:hover {
  background: rgba(255, 107, 107, 0.2);
}

.custom-dropdown-divider {
  height: 1px;
  background: #2a2a4e;
  margin: 4px 0;
}

.new-board-item {
  color: #0d6efd;
  font-weight: 500;
}

.new-board-item:hover {
  background: rgba(13, 110, 253, 0.1);
}

/* Modal Styles */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 3000;
  backdrop-filter: blur(5px);
}

.modal-content {
  background: #1a1a2e;
  border: 1px solid #2a2a4e;
  border-radius: 12px;
  width: 90%;
  max-width: 700px;
  max-height: 85vh;
  overflow-y: auto;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 24px;
  border-bottom: 1px solid #2a2a4e;
}

.modal-title {
  color: white;
  margin: 0;
  font-size: 1.5rem;
}

.btn-close-custom {
  background: none;
  border: none;
  color: #888;
  font-size: 28px;
  cursor: pointer;
  padding: 0;
  line-height: 1;
}

.btn-close-custom:hover {
  color: white;
}

.btn-close-custom:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.modal-body {
  padding: 24px;
}

/* Board Modal Specific */
.board-modal .create-board-section,
.board-modal .import-board-section,
.board-modal .existing-boards-section {
  margin-bottom: 24px;
}

.board-modal h4 {
  color: #ccc;
  margin-bottom: 12px;
  font-size: 1.1rem;
}

.board-modal .input-group {
  display: flex;
  gap: 8px;
}

.board-modal .form-control {
  background: #2a2a4e;
  border: 1px solid #3a3a6e;
  color: white;
}

.board-modal .form-control:focus {
  background: #3a3a6e;
  border-color: #4a4a8e;
  color: white;
  box-shadow: none;
}

.board-modal .form-control::placeholder {
  color: #666;
}

/* Boards List */
.boards-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  max-height: 300px;
  overflow-y: auto;
}

.board-card {
  background: #2a2a4e;
  border: 1px solid #3a3a6e;
  border-radius: 8px;
  padding: 12px 16px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  transition: all 0.2s;
  cursor: pointer;
}

.board-card:hover {
  background: #3a3a6e;
  border-color: #4a4a8e;
}

.board-card.active {
  border-color: #0d6efd;
  background: #1a2744;
}

.board-card-info {
  flex: 1;
}

.board-card-name {
  color: white;
  font-weight: 500;
  margin-bottom: 4px;
}

.board-card-meta {
  display: flex;
  gap: 16px;
}

.board-card-meta small {
  color: #888;
  font-size: 0.75rem;
}

.board-card-actions {
  display: flex;
  gap: 4px;
  opacity: 0;
  transition: opacity 0.2s;
}

.board-card:hover .board-card-actions {
  opacity: 1;
}

/* No Boards Message */
.no-boards-message {
  padding: 32px 0;
}

.no-boards-message h5 {
  color: #ccc;
}

/* Navbar Improvements */
.navbar .vr {
  background-color: #444;
  width: 1px;
  height: 20px;
}

.navbar .btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}


/* Trello Sidebar Labels */
.trello-sidebar-labels-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.trello-sidebar-label-item {
  display: flex;
  align-items: center;
  gap: 8px;
}

.trello-sidebar-label-visual {
  min-width: 40px;
}

.trello-sidebar-label-info {
  flex: 1;
}

.trello-sidebar-label-input {
  background: #2a2a4e !important;
  border: 1px solid #3a3a6e !important;
  color: white !important;
}
</style>