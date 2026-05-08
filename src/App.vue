<template>
  <section class="h-100 d-flex flex-column" :class="{ 'dark-mode': isDarkMode }">
    <!-- Navigation Bar -->
    <nav class="navbar navbar-dark" :class="{ 'bg-dark': isDarkMode, 'bg-light': !isDarkMode }">
      <div class="container-fluid flex-nowrap overflow-auto">
        <div class="d-flex align-items-center me-auto">

          <button @click="openBoardOverlay" class="btn btn-sm btn-outline-info me-2 text-nowrap">
            <i class="fas fa-clipboard-list"></i> {{ boards.length > 0 ? 'Boards' : 'Create Board' }}
          </button>

          <div class="vr me-2"></div>

          <label for="fileInput" class="btn btn-sm btn-outline-primary me-2 text-nowrap">
            <i class="fas fa-folder-open"></i> Upload
          </label>
          <input type="file" id="fileInput" @change="onFileChange" hidden accept=".json">

          <button @click="downloadData" class="btn btn-sm btn-outline-success me-2 text-nowrap"
            :disabled="!activeBoardId">
            <i class="fas fa-download"></i> Download
          </button>
          <button @click="SaveLocalStorage" class="btn btn-sm btn-outline-info me-2 text-nowrap"
            :disabled="!activeBoardId">
            <i class="fas fa-save"></i> Save
          </button>
          <button @click="resetEverything" class="btn btn-sm btn-outline-warning me-2 text-nowrap">
            <i class="fas fa-sync-alt"></i> Reset All
          </button>

          <div class="vr mx-2"></div>

          <div class="form-check form-switch d-flex align-items-center ms-2 text-nowrap">
            <input type="checkbox" v-model="autoSaveCheckbox" class="form-check-input" id="autoSaveCheckbox" />
            <label class="form-check-label ms-2" :class="{ 'text-light': isDarkMode, 'text-dark': !isDarkMode }"
              for="autoSaveCheckbox" style="font-size: 0.85rem;">
              <i class="fas fa-magic"></i> Auto Save
            </label>
          </div>
          <div class="vr mx-2"></div>
          <div class="form-check form-switch d-flex align-items-center ms-2 text-nowrap">
            <input type="checkbox" v-model="labelNameCheckbox" class="form-check-input" id="labelNameCheckbox" />
            <label class="form-check-label ms-2" :class="{ 'text-light': isDarkMode, 'text-dark': !isDarkMode }"
              for="labelNameCheckbox" style="font-size: 0.85rem;">
              <i class="fas fa-eye-slash"></i> Hide Labels
            </label>
          </div>
          <div class="vr mx-2"></div>
          <div class="form-check form-switch d-flex align-items-center ms-2 text-nowrap">
            <input type="checkbox" v-model="closedCheckbox" class="form-check-input" id="closedSwitch" />
            <label class="form-check-label ms-2" :class="{ 'text-light': isDarkMode, 'text-dark': !isDarkMode }"
              for="closedSwitch" style="font-size: 0.85rem;">
              <i class="fas fa-archive"></i> Archived
            </label>
          </div>
        </div>

        <div class="d-flex align-items-center ms-3">
          <!-- Dark Mode Toggle -->
          <button @click="toggleDarkMode" class="btn btn-sm me-2 text-nowrap"
            :class="{ 'btn-outline-light': isDarkMode, 'btn-outline-dark': !isDarkMode }"
            :title="isDarkMode ? 'Switch to Light Mode' : 'Switch to Dark Mode'">
            <i :class="isDarkMode ? 'fas fa-sun' : 'fas fa-moon'"></i>
          </button>

          <div class="btn-group">
            <button @click="zoomIn" class="btn btn-sm btn-outline-secondary text-nowrap" title="Zoom In">
              <i class="fas fa-search-plus"></i>
            </button>
            <button @click="zoomOut" class="btn btn-sm btn-outline-secondary text-nowrap" title="Zoom Out">
              <i class="fas fa-search-minus"></i>
            </button>
            <button @click="resetZoom" class="btn btn-sm btn-outline-secondary text-nowrap" title="Reset Zoom">
              <i class="fas fa-expand"></i> {{ zoomLevel }}%
            </button>
          </div>
        </div>

      </div>
    </nav>

    <!-- Board Overlay Modal (Board manager) -->
    <div v-if="showBoardOverlay" class="modal-overlay" @click.self="closeBoardOverlay()">
      <div class="modal-content board-modal" :class="{ 'dark-mode': isDarkMode }" @click.stop>
        <div class="modal-header">
          <h3 class="modal-title"><i class="fas fa-clipboard-list"></i> Board Management</h3>
          <button v-if="activeBoardId" @click="closeBoardOverlay" class="btn-close-custom" title="Close">
            <i class="fas fa-times"></i>
          </button>
        </div>

        <div class="modal-body">
          <!-- Create New Board Section -->
          <div class="create-board-section mb-4">
            <h4><i class="fas fa-plus-circle"></i> Create New Board</h4>
            <div class="input-group">
              <input v-model="newBoardName" type="text" placeholder="Enter board name..." class="form-control"
                @keyup.enter="createNewBoard" ref="newBoardInput" />
              <button @click="createNewBoard" class="btn btn-primary" :disabled="!newBoardName.trim()">
                <i class="fas fa-plus"></i> Create
              </button>
            </div>
          </div>

          <!-- Import Board Section -->
          <div class="import-board-section mb-4">
            <h4><i class="fas fa-file-import"></i> Import Board from File</h4>
            <div class="input-group">
              <input type="file" @change="importBoardFromFile" class="form-control" accept=".json"
                ref="importFileInput" />
            </div>
          </div>

          <!-- Existing Boards List -->
          <div v-if="boards.length > 0" class="existing-boards-section">
            <h4><i class="fas fa-list"></i> Your Boards ({{ boards.length }})</h4>
            <div class="boards-list">
              <div v-for="board in boards" :key="board.id" class="board-card"
                @click="switchBoard(board.id); closeBoardOverlay()" :class="{ active: board.id === activeBoardId }">
                <div class="board-card-info">
                  <div class="board-card-name">
                    <i class="fas fa-clipboard"></i> {{ board.name }}
                  </div>
                  <div class="board-card-meta">
                    <small><i class="far fa-calendar-plus"></i> Created: {{ formatDate(board.createdAt) }}</small>
                    <small><i class="far fa-calendar-check"></i> Modified: {{ formatDate(board.lastModified) }}</small>
                  </div>
                </div>
                <div class="board-card-actions">
                  <button @click.stop="switchBoard(board.id); closeBoardOverlay()" class="btn btn-sm btn-outline-light"
                    title="Open Board">
                    <i class="fas fa-folder-open"></i>
                  </button>
                  <button @click.stop="deleteBoard(board.id)" class="btn btn-sm btn-outline-danger"
                    title="Delete Board">
                    <i class="fas fa-trash-alt"></i>
                  </button>
                </div>
              </div>
            </div>
          </div>

          <!-- No Boards Message -->
          <div v-else class="no-boards-message">
            <div class="text-center p-4">
              <div class="fs-1 mb-3"><i class="fas fa-rocket"></i></div>
              <h5>No Boards Yet</h5>
              <p class="text-muted">Create a new board or import one from a file to get started!</p>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Sidebar Toggle Button -->
    <button class="sidebar-toggle" @click="toggleSidebar" :class="{ active: sidebarOpen }">
      <span v-if="!sidebarOpen"><i class="fas fa-th-large"></i></span>
      <span v-else><i class="fas fa-times"></i></span>
    </button>

    <!-- Sidebar -->
    <div class="sidebar" :class="{ open: sidebarOpen }">
      <!-- Tab Navigation -->
      <div class="sidebar-tabs">
        <button v-for="tab in sidebarTabs" :key="tab.id" class="sidebar-tab" :class="{ active: activeTab === tab.id }"
          @click="activeTab = tab.id">
          <i :class="tab.icon"></i> {{ tab.label }}
        </button>
      </div>

      <!-- Tab Content -->
      <div class="sidebar-content">
        <!-- Main Tab -->
        <div v-if="activeTab === 'main' && $refs.viewerRef && $refs.viewerRef.main" class="tab-panel">
          <h5><i class="fas fa-info-circle"></i> Board Info</h5>
          <div class="sidebar-field">
            <div>
              <label><i class="fas fa-tag"></i> Board Name:</label>
              <input v-model="$refs.viewerRef.main.name" type="text" class="form-control" @change="onDataChanged" />
            </div>
          </div>
          <!-- Board Stats Table -->
          <div class="board-stats-table" v-if="$refs.viewerRef && $refs.viewerRef.main">
            <h5 class="mb-3">📊 Board Statistics</h5>

            <table class="stats-table">
              <tbody>
                <tr v-if="$refs.viewerRef.main.lists">
                  <td class="stats-icon">
                    <span class="icon-circle bg-primary">
                      <i class="fas fa-list-ul"></i>
                    </span>
                  </td>
                  <td class="stats-label">Lists</td>
                  <td class="stats-value">
                    <span class="badge bg-primary">{{ $refs.viewerRef.main.lists.length ?? 0 }}</span>
                  </td>
                </tr>

                <tr v-if="$refs.viewerRef.main.cards">
                  <td class="stats-icon">
                    <span class="icon-circle bg-success">
                      <i class="far fa-credit-card"></i>
                    </span>
                  </td>
                  <td class="stats-label">Cards</td>
                  <td class="stats-value">
                    <span class="badge bg-success">{{ $refs.viewerRef.main.cards.length ?? 0 }}</span>
                  </td>
                </tr>

                <tr v-if="$refs.viewerRef.main.checklists">
                  <td class="stats-icon">
                    <span class="icon-circle bg-info">
                      <i class="fas fa-tasks"></i>
                    </span>
                  </td>
                  <td class="stats-label">Checklists</td>
                  <td class="stats-value">
                    <span class="badge bg-info">{{ $refs.viewerRef.main.checklists.length ?? 0 }}</span>
                  </td>
                </tr>

                <tr v-if="$refs.viewerRef.main.exiting_tags">
                  <td class="stats-icon">
                    <span class="icon-circle bg-warning">
                      <i class="fas fa-tags"></i>
                    </span>
                  </td>
                  <td class="stats-label">Tags</td>
                  <td class="stats-value">
                    <span class="badge bg-warning">{{ $refs.viewerRef.main.exiting_tags.length ?? 0 }}</span>
                  </td>
                </tr>
              </tbody>
            </table>

            <div class="stats-divider"></div>

            <h5 class="mb-3">📅 Activity</h5>

            <table class="stats-table">
              <tbody>
                <tr v-if="$refs.viewerRef.main.dateLastActivity">
                  <td class="stats-icon">
                    <span class="icon-circle bg-secondary">
                      <i class="fas fa-history"></i>
                    </span>
                  </td>
                  <td class="stats-label">Last Activity</td>
                  <td class="stats-value date-value">
                    {{ formatDate($refs.viewerRef.main.dateLastActivity) }}
                  </td>
                </tr>

                <tr v-if="$refs.viewerRef.main.dateLastView">
                  <td class="stats-icon">
                    <span class="icon-circle bg-secondary">
                      <i class="far fa-eye"></i>
                    </span>
                  </td>
                  <td class="stats-label">Last Viewed</td>
                  <td class="stats-value date-value">
                    {{ formatDate($refs.viewerRef.main.dateLastView) }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

        </div>

        <!-- Data Tab -->
        <div v-if="activeTab === 'data'" class="tab-panel">
          <h5><i class="fas fa-database"></i> Data Management</h5>
          <button @click="SaveLocalStorage" class="btn btn-sm btn-outline-info w-100 mb-2" :disabled="!activeBoardId">
            <i class="fas fa-save"></i> Save Current Board
          </button>
          <button @click="downloadData" class="btn btn-sm btn-outline-success w-100 mb-2" :disabled="!activeBoardId">
            <i class="fas fa-download"></i> Download as JSON
          </button>
          <button @click="openBoardOverlay" class="btn btn-sm btn-outline-primary w-100 mb-2">
            <i class="fas fa-clipboard-list"></i> Manage Boards
          </button>
          <hr>
          <button @click="resetEverything" class="btn btn-sm btn-outline-danger w-100">
            <i class="fas fa-sync-alt"></i> Reset Everything
          </button>
        </div>

        <!-- Label Tab -->
        <div v-if="activeTab === 'label' && $refs.viewerRef && $refs.viewerRef.main" class="tab-panel">
          <h5><i class="fas fa-tags"></i> Default Labels</h5>
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
          <div v-else class="text-muted">
            <i class="fas fa-exclamation-circle"></i> No labels found
          </div>
        </div>
      </div>
    </div>

    <!-- Sidebar Overlay -->
    <div v-if="sidebarOpen" class="sidebar-overlay" @click="toggleSidebar"></div>

    <!-- Main Viewer Component -->
    <Viewer ref="viewerRef" :labelColorMap="labelColorMap" :export_data="fileText" :is_archived="closedCheckbox" :darkMode="isDarkMode" :labelNameCheckbox="labelNameCheckbox" />

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
      isDarkMode: false,

      fileText: '',
      fileBinary: null,

      closedCheckbox: false,
      autoSaveCheckbox: true,
      labelNameCheckbox: true,

      zoomLevel: 100,

      sidebarOpen: false,
      activeTab: 'main',
      sidebarTabs: [
        { id: 'main', label: 'Board Info', icon: 'fas fa-info-circle' },
        { id: 'data', label: 'Data', icon: 'fas fa-database' },
        { id: 'label', label: 'Labels', icon: 'fas fa-tags' }
      ],

      // Board Management
      boards: [],
      activeBoardId: null,
      showBoardOverlay: true, // Default to open on startup

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
    this.initDarkMode();

    this.loadPreferences();
    this.loadBoards();

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
      this.showBoardOverlay = false;
    }

    if (this.autoSaveCheckbox) {
      this.startAutoSave();
    }
  },
  beforeUnmount() {
    this.stopAutoSave();
    this.saveCurrentBoardData();
  },
  methods: {
    toggleDarkMode() {
      this.isDarkMode = !this.isDarkMode;

      // Save preference to localStorage
      localStorage.setItem('trello-dark-mode', this.isDarkMode);

      // Apply to body for any global styles
      if (this.isDarkMode) {
        document.body.classList.add('dark-mode');
      } else {
        document.body.classList.remove('dark-mode');
      }
    },

    /**
     * Initialize dark mode from saved preference
     */
    initDarkMode() {
      const savedMode = localStorage.getItem('trello-dark-mode');
      if (savedMode !== null) {
        this.isDarkMode = savedMode === 'true';
      }

      // Apply initial state
      if (this.isDarkMode) {
        document.body.classList.add('dark-mode');
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
      this.newBoardName = '';
      this.$nextTick(() => {
        if (this.$refs.newBoardInput) {
          this.$refs.newBoardInput.focus();
        }
      });
    },

    closeBoardOverlay() {
      if (this.activeBoard != null) {
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

    // Update createNewBoard
    createNewBoard() {
      const name = this.newBoardName.trim();
      if (!name) return;

      const boardId = this.generateBoardId();
      const now = new Date().toISOString();

      const newBoard = {
        id: boardId,
        name: name,
        createdAt: now,
        lastModified: now
      };

      this.boards.push(newBoard);
      this.saveBoards();

      // Create board data with just the basic structure
      const boardData = this.createEmptyBoard(boardId, name)

      // Save to localStorage
      localStorage.setItem(`board_${boardId}`, JSON.stringify(boardData));

      // Switch to the new board
      this.switchBoard(boardId);

      // Reset input
      this.newBoardName = '';
    },

    // Update loadBoard - simplified
    loadBoard(boardId) {
      const boardData = localStorage.getItem(`board_${boardId}`);
      const board = this.boards.find(b => b.id === boardId);

      if (boardData) {
        // Just pass the raw JSON string to the viewer
        this.fileText = boardData;
        this.activeBoardId = boardId;
        localStorage.setItem('lastActiveBoardId', boardId);

        if (board) {
          board.lastViewed = new Date().toISOString();
          this.saveBoards();
        }
      } else {
        // Create new empty data
        const emptyData = this.createEmptyBoard(boardId, board ? board.name : 'New Board');
        const dataString = JSON.stringify(emptyData);
        localStorage.setItem(`board_${boardId}`, dataString);
        this.fileText = dataString;
        this.activeBoardId = boardId;
        localStorage.setItem('lastActiveBoardId', boardId);
      }
    },

    createEmptyBoard(boardId, name) {
      return {
        id: boardId,
        name: name,
        dateLastActivity: new Date().toISOString(),
        dateLastView: new Date().toISOString(),
        labelNames: { black: '', black_dark: '', black_light: '', blue: '', blue_dark: '', blue_light: '', green: '', green_dark: '', green_light: '', lime: '', lime_dark: '', lime_light: '', orange: '', orange_dark: '', orange_light: '', pink: '', pink_dark: '', pink_light: '', purple: '', purple_dark: '', purple_light: '', red: '', red_dark: '', red_light: '', sky: '', sky_dark: '', sky_light: '', yellow: '', yellow_dark: '', yellow_light: '', Done: '' },
        checklists: [],
        cards: [],
        lists: [],
        actions: [],
        exiting_tags: []
      };
    },

    // Update saveCurrentBoardData - simplified
    saveCurrentBoardData() {
      if (this.activeBoardId && this.$refs.viewerRef && this.$refs.viewerRef.main) {
        try {
          const currentData = this.$refs.viewerRef.main;

          // Sync board name from main data
          if (currentData.name) {
            const board = this.boards.find(b => b.id === this.activeBoardId);
            if (board) {
              board.name = currentData.name;
              board.lastModified = new Date().toISOString();
              this.saveBoards();
            }
          }

          // Save the complete data
          const dataString = JSON.stringify(currentData);
          localStorage.setItem(`board_${this.activeBoardId}`, dataString);
        } catch (error) {
          console.error('Error saving board:', error);
        }
      }
    },

    // Update switchBoard
    switchBoard(boardId) {
      if (boardId === this.activeBoardId) return;

      // Save current board first
      this.saveCurrentBoardData();

      // Load new board
      this.loadBoard(boardId);
    },

    // Update importBoardFromFile
    importBoardFromFile(event) {
      const file = event.target.files[0];
      if (!file) return;

      const reader = new FileReader();
      reader.onload = (e) => {
        try {
          const boardData = JSON.parse(e.target.result);
          const boardName = boardData.name || file.name.replace('.json', '');
          const boardId = this.generateBoardId();
          const now = new Date().toISOString();

          const newBoard = {
            id: boardId,
            name: boardName || 'Imported Board',
            createdAt: now,
            lastModified: now
          };

          // Check storage size
          const dataString = JSON.stringify(boardData);
          if (dataString.length > 4 * 1024 * 1024) {
            alert('Warning: This board is very large (over 4MB). It may cause storage issues.');
          }

          this.boards.push(newBoard);
          this.saveBoards();

          // Save the imported data
          try {
            // Ensure the data has id and name
            boardData.id = boardId;
            boardData.name = boardName;
            localStorage.setItem(`board_${boardId}`, JSON.stringify(boardData));
          } catch (storageError) {
            this.boards = this.boards.filter(b => b.id !== boardId);
            this.saveBoards();
            alert('Error: Not enough storage space. Try clearing some old boards first.');
            return;
          }

          // Switch to the new board
          this.switchBoard(boardId);

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
