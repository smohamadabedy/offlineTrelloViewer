<template>
  <div class="scroll-container m-0 pt-5" @mousedown="startDrag" @mousemove="onDrag" @mouseup="stopDrag"
    @mouseleave="stopDrag">

    <div id="zoom-canvas" class="scroll-row px-3">

      <!-- ========== LISTS ========== -->
      <template v-for="(list, index) in main.lists" :key="index">
        <div class="list-item" v-if="list.closed == is_archived" :data-list-index="index" :data-list-id="list.id"
          :class="{ 'dragging-source': draggedListIndex === index, 'drag-over-list': draggedOverListId === list.id && draggedCardSourceList !== list.id }">

          <!-- LIST HEADER -->
          <div class="header" draggable="true" @dragstart="onListDragStart($event, index)" @dragend="onListDragEnd"
            @dragover="onListDragOver($event, index)" @drop="onListDrop($event, index)">

            <div class="header-colorbar" v-if="list.color != null" :style="{ 'background-color': list.color }"></div>
            <div class="drag-handle">
              <small><i class="fas fa-bars"></i></small>
            </div>

            <div class="list-title" v-if="renamingIndex !== index">
              <span @click="startRename(index, $event)" class="cursor-text">{{ list.name }}</span>
            </div>
            <input v-else v-model="renameValue" @click.stop @change="saveRename(index)" @keyup.esc="cancelRename"
              @mousedown.stop class="rename-input" ref="renameInput" autofocus />

            <div class="header-dots">
              <button class="btn btn-sm header-dots-btn" @click.stop="toggleSettingsPanel(index)">
                <i class="fas fa-ellipsis-v"></i>
              </button>

              <!-- Settings Panel -->
              <div v-if="activeSettingsPanel === index" class="settings-panel">
                <div class="settings-item">
                  <button @click="startRename(index, $event)">
                    <i class="fas fa-pencil-alt fa-sm"></i>
                    <small> Rename</small>
                  </button>
                </div>
                <div class="settings-item  settings-item-row">
                  <input type="color" v-model="list.color" class="btn color-custom" title="Custom color" />
                  <button @click="removeColor(index)" class="p-2">
                    <i class="fas fa-times"></i>
                  </button>
                </div>
                <div class="settings-item settings-item-row">
                  <button @click="addListBefore(index)">
                    <i class="fas fa-arrow-left"></i>
                    <small><br />Add</small>
                  </button>
                  <button @click="addListAfter(index)">
                    <i class="fas fa-arrow-right"></i>
                    <small><br />Add</small>
                  </button>
                </div>
                <div class="settings-item settings-item-row">
                  <button @click="moveListUp(index)">
                    <i class="fas fa-arrow-left"></i>
                    <small><br />Move</small>
                  </button>
                  <button @click="moveListDown(index)">
                    <i class="fas fa-arrow-right"></i>
                    <small><br />Move</small>
                  </button>
                </div>
                <div class="settings-item">
                  <button @click="archiveList(index)">
                    <i class="fas fa-archive"></i>
                    <small> Archive</small>
                  </button>
                </div>
                <div class="settings-item">
                  <button @click="deleteList(index)" class="danger-btn">
                    <i class="fas fa-trash-alt"></i>
                    <small> Delete</small>
                  </button>
                </div>
              </div>
            </div>

            <!-- Card Dropdown Menu -->
            <div v-if="cardMenuExpand == index" class="card-dropdown" @click.stop>
              <button class="card-dropdown-item" @click.stop="openLabelPopup(this.cardDataforMenu)">
                <i class="fas fa-tags fa-sm"></i> Edit Labels
              </button>
              <button class="card-dropdown-item" @click.stop="moveCardUp(this.cardDataforMenu)">
                <i class="fas fa-arrow-up fa-sm"></i> Move Up
              </button>
              <button class="card-dropdown-item" @click.stop="moveCardDown(this.cardDataforMenu)">
                <i class="fas fa-arrow-down fa-sm"></i> Move Down
              </button>
              <button class="card-dropdown-item" @click.stop="moveCardTop(this.cardDataforMenu)">
                <i class="fas fa-angle-double-up fa-sm"></i> Move Top
              </button>
              <button class="card-dropdown-item" @click.stop="moveCardBottom(this.cardDataforMenu)">
                <i class="fas fa-angle-double-down fa-sm"></i> Move Bottom
              </button>
              <button class="card-dropdown-item" @click.stop="archiveCardFromMenu(this.cardDataforMenu)">
                <i class="fas fa-archive fa-sm"></i> Archive
              </button>
            </div>

          </div>

          <!-- CONTENT -->
          <div class="content" @dragover="onCardContentDragOver($event, list.id)"
            @dragleave="onCardContentDragLeave($event, list.id)" @drop="onCardDrop($event, list.id)">

            <!-- ===== CARD ===== -->
            <div v-for="(card, cardIndex) in getCardsByList(list.id)" :key="card.id" class="card"
              :data-card-id="card.id" :data-card-index="cardIndex" :data-source-list="list.id" :class="{
                'card-dragging': draggedCard && draggedCard.id === card.id,
                'drag-over-card': dragOverTarget && dragOverTarget.cardId === card.id && dragOverTarget.position === 'before',
                'drag-over-card-after': dragOverTarget && dragOverTarget.cardId === card.id && dragOverTarget.position === 'after'
              }" draggable="true" @dragstart="onCardDragStart($event, card, cardIndex, list.id)"
              @dragend="onCardDragEnd" @dragover="onCardDragOverCard($event, card, cardIndex, list.id)"
              @dragleave="onCardDragLeaveCard($event, card, cardIndex, list.id)"
              @drop="onCardDropOnCard($event, card, cardIndex, list.id)" @click.stop="openCardPopup(card.id)"
              @contextmenu.prevent="toggleCardMenu($event, card.id, index, card)">

              <!-- Card labels and name -->
              <div class="card-title">
                <template v-for="(item, idx) in card.labels">
                  <LabelBar class="mb-2" :labelKey="card.labels[idx].color" :show-none="labelNameCheckbox"
                    :show-label="true"
                    :display-value="card.labels[idx].name && card.labels[idx].name.length > 0 ? card.labels[idx].name : main.labelNames[card.labels[idx].color]"
                    :labelColorMap="labelColorMap" />
                </template>
                <span>{{ card.name }}</span>
              </div>

              <!-- Card indicators - bottom left -->
              <div class="card-indicators" v-if="hasCardIndicators(card)">
                <!-- Due date icon -->
                <span v-if="card.dueDate" class="card-indicator">
                  <i class="far fa-clock"></i>
                </span>

                <!-- Description icon -->
                <span v-if="card.desc && card.desc.trim().length > 0" class="card-indicator">
                  <i class="fas fa-align-left"></i>
                </span>

                <!-- Checklist icon -->
                <span v-if="hasChecklists(card)" class="card-indicator">
                  <i class="fas fa-check-square"></i>
                </span>
              </div>

              <!-- Vertical three-dot button + dropdown -->
              <div class="card-actions-wrapper">
                <button class="card-menu-btn btn btn-sm" @click.stop="toggleCardMenu($event, card.id, index, card)"
                  title="Card actions">
                  <i class="fas fa-ellipsis-v"></i>
                </button>
              </div>
            </div>

            <div v-if="getCardsByList(list.id).length === 0" class="empty-list-hint">
              Drop cards here
            </div>
          </div>

          <!-- LIST FOOTER -->
          <div class="footer">
            <div class="footer-div"></div>
            <div v-if="!addingCardToListId || addingCardToListId !== list.id" class="add-card-button"
              @click.stop="showAddCardInput(list.id)">
              <i class="fas fa-plus fa-xs"></i> Add a card
            </div>
            <div v-else class="add-card-form" @click.stop>
              <input ref="cardInput" v-model="newCardName" type="text" class="add-card-input"
                placeholder="Enter card title..." @keyup.enter="addCard(list.id)" @keyup.escape="cancelAddCard" />
              <div class="add-card-actions">
                <button class="add-card-submit" @click="addCard(list.id)" :disabled="!newCardName.trim()">
                  <small><i class="fas fa-check fa-sm"></i> Add Card</small>
                </button>
                <button class="add-card-cancel" @click="cancelAddCard">
                  <small><i class="fas fa-times"></i></small>
                </button>
              </div>
            </div>
          </div>
        </div>
      </template>

      <!-- Add new list -->
      <div class="list-item add-list-item">
        <div v-if="!isAddingList" class="add-list-button" @click="showAddListInput">
          <i class="fas fa-plus fa-xs"></i> Add another list
        </div>
        <div v-else class="add-list-form" @click.stop>
          <input ref="listInput" v-model="newListName" type="text" class="add-list-input"
            placeholder="Enter list title..." @keyup.enter="addList(null)" @keyup.escape="cancelAddList" />
          <div class="add-list-actions">
            <button class="add-list-submit" @click="addList(null)" :disabled="!newListName.trim()">
              <small><i class="fas fa-check fa-sm"></i> Add List</small>
            </button>
            <button class="add-list-cancel" @click="cancelAddList">
              <small><i class="fas fa-times"></i></small>
            </button>
          </div>
        </div>
      </div>

      <div>&nbsp;</div>
    </div>

    <!-- ========== LABEL POPUP ========== -->
    <div v-if="showLabelPopup" class="popup-overlay" @click="closeLabelPopup">
      <div class="popup-content label-popup" @click.stop>
        <div class="popup-header">
          <h2><i class="fas fa-tags"></i> Labels – {{ labelPopupCard.name }}</h2>
          <button class="close-btn" @click="closeLabelPopup">
            <i class="fas fa-times"></i>
          </button>
        </div>
        <div class="popup-body">
          <!-- Current labels -->
          <div v-if="labelPopupCard.labels && labelPopupCard.labels.length > 0" class="mb-3">
            <label class="form-label" style="color: var(--text-secondary);">Current labels</label>
            <div class="label-chip-list">
              <div v-for="label in labelPopupCard.labels" :key="label.id" class="label-chip"
                :style="{ backgroundColor: labelColorMap[label.color] || label.color }">
                <LabelBar v-if="label.name && label.name.length > 0" :labelKey="label.color" :show-none="false"
                  :show-label="true" :display-value="label.name" :labelColorMap="labelColorMap" />
                <LabelBar v-else :labelKey="label.color" :show-label="true"
                  :display-value="main.labelNames[label.color] && main.labelNames[label.color].length > 0 ? main.labelNames[label.color] : label.color"
                  :show-none="false" :labelColorMap="labelColorMap" />
                <button class="btn-close-label" @click.stop="removeLabelFromCard(label.id)">
                  <i class="fas fa-times fa-xs"></i>
                </button>
              </div>
            </div>
          </div>

          <!-- Add new label -->
          <hr style="border-color: var(--border-primary);" />
          <div class="add-label-section">
            <label class="form-label" style="color: var(--text-secondary);">Pick a color</label>
            <div class="color-palette mb-4">
              <div v-for="(hex, key) in labelColorMap" :key="key" class="color-swatch"
                :class="{ selected: selectedLabelColor === key }" :style="{ backgroundColor: hex }" :title="key"
                @click="selectedLabelColor = key">
                <LabelBar @click="selectedLabelColor = key" class="mb-2" :labelKey="key" :show-none="true"
                  :version-small="true" :labelColorMap="labelColorMap" />
              </div>
              <button v-if="selectedLabelColor" class="btn btn-sm btn-outline-secondary ms-2"
                @click="selectedLabelColor = null">
                <i class="fas fa-times"></i> Clear
              </button>
            </div>
            <label class="form-label" style="color: var(--text-secondary);">Optional: Add a name</label>
            <div class="mb-2">
              <input v-model="newLabelName" type="text" class="form-control" placeholder="Label name..."
                style="background: var(--bg-input); color: var(--text-primary); border-color: var(--border-primary);"
                @keyup.enter="addLabelToCard" />
            </div>
          </div>
        </div>
        <div class="popup-footer" style="padding: 16px; border-top: 1px solid var(--border-primary);">
          <button class="btn btn-primary btn-block w-100" @click="addLabelToCard">
            <i class="fas fa-plus fa-sm"></i> Add
          </button>
        </div>
      </div>
    </div>

    <!-- ========== CARD DETAIL POPUP ========== -->
    <div v-if="showPopup" class="popup-overlay" @click="closePopup">
      <div class="popup-content" @click.stop>
        <!-- Fixed Header -->
        <div class="popup-header">
          <h2><span>{{ getListName(selectedCard.idList) }}</span> <i class="fas fa-long-arrow-alt-right"></i> {{
            selectedCard.name }}</h2>
          <button class="close-btn" @click="closePopup">
            <i class="fas fa-times"></i>
          </button>
        </div>

        <!-- Scrollable Body -->
        <div class="popup-body">
          <div class="card-dates-row mb-3 p-3" style="background: var(--bg-secondary); border-radius: 8px;">
            <div class="popup-field mb-3">
              <div class="input-group mb-3">
                <span class="input-group-text"
                  style="background: var(--bg-input); border-color: var(--border-primary); color: var(--text-tertiary);">
                  <i class="fas fa-pencil-alt fa-sm"></i>
                </span>
                <input class="form-control p-2" type="text" v-model="selectedCard.name" placeholder="Card name..."
                  style="background: var(--bg-input); color: var(--text-primary); border-color: var(--border-primary); font-weight: 500;" />
              </div>

            </div>
            <div class="row g-3">
              <div class="col-6">
                <small style="color: var(--text-tertiary); display: block;"><i class="far fa-calendar-plus"></i>
                  Created</small>
                <span style="color: var(--text-primary); font-size: 0.9rem;">
                  {{ formatDate(selectedCard.createdAt || null) }}
                </span>
              </div>
              <div class="col-6">
                <small style="color: var(--text-tertiary); display: block;"><i class="fas fa-edit"></i> Last
                  Modified</small>
                <span style="color: var(--text-primary); font-size: 0.9rem;">
                  {{ formatDate(selectedCard.dateLastActivity) }}
                </span>
              </div>
              <div class="col-md-4" v-if="selectedCard.closed && selectedCard.dateClosed">
                <small style="color: var(--text-tertiary); display: block;"><i class="fas fa-archive"></i>
                  Closed</small>
                <span style="color: var(--text-primary); font-size: 0.9rem;">
                  {{ formatDate(selectedCard.dateClosed) }}
                </span>
              </div>
            </div>

            <div class="popup-field mt-3 mb-3">
              <vue-tags-input class="tags-input-dark" :value="this.selectedCard.tags"
                :existing-tags="this.main.exiting_tags" placeholder="Search or add tag..." :typeahead="true" :limit="10"
                :typeahead-max-results="4" typeahead-style="dropdown"
                @tag-added="slug => handleTagsChanged(slug, selectedCard)" />
            </div>
          </div>


          <div class="popup-field">
            <label><i class="fas fa-align-left fa-sm"></i> Description:</label>
                <MdEditor ref="editorRef" v-model="selectedCard.desc" :theme="darkMode ? 'dark' : 'light'" />
          </div>

          <div class="popup-field">
            <label><i class="far fa-calendar-alt"></i> Due Date:</label>
            <div class="row g-2 align-items-center">
              <div class="col-6 col-md-8">
                <p class="due-date-text mb-0" style="color: var(--text-secondary);">{{ selectedCard.dueDate || 'No due date' }}</p>
              </div>
              <div class="col-6 col-md-4">
                <input type="date" v-model="selectedCard.dueDate" class="date-picker-input form-control"
                  :class="{ 'no-date': !selectedCard.dueDate }"
                  :data-placeholder="selectedCard.dueDate ? '' : 'No due date'" />
              </div>
            </div>
          </div>

          <div class="popup-field">
            <label><i class="fas fa-check-square"></i> Checklists:</label>

            <!-- Existing checklists -->
            <div v-for="checklist in getCardChecklists(selectedCard.idChecklists)" :key="checklist.id"
              class="checklist">

              <!-- Checklist header -->
              <div class="checklist-header">
                <input v-model="checklist.name" class="checklist-title-input" placeholder="Checklist title..." />
                <button class="btn btn-sm btn-danger-outline" @click.stop="deleteChecklist(checklist.id)">
                  <i class="fas fa-trash-alt"></i>
                </button>
              </div>

              <!-- Progress bar -->
              <div class="checklist-progress small text-muted mb-1" v-if="checklist.checkItems.length"
                style="color: var(--text-tertiary) !important;">
                {{ getCompletedCount(checklist) }}/{{ checklist.checkItems.length }} done
              </div>

              <!-- Items -->
              <div class="checklist-items">
                <div v-for="(item, itemIdx) in checklist.checkItems" :key="item.id" class="checklist-item">

                  <!-- Checkbox -->
                  <input type="checkbox" :checked="item.state === 'complete'" @change="toggleItemState(checklist, item)"
                    :id="item.id" class="checklist-checkbox" />

                  <!-- Item name (inline editing) -->
                  <span v-if="editingItemId !== item.id" class="checklist-item-name" @click="startEditItem(item)">
                    {{ item.name }}
                  </span>
                  <input v-else v-model="editItemName" class="checklist-item-input"
                    @keyup.enter="saveEditItem(checklist, item)" @keyup.escape="cancelEditItem"
                    @blur="saveEditItem(checklist, item)" ref="itemEditInput" autofocus />

                  <!-- Item controls -->
                  <div class="checklist-item-actions">
                    <button class="btn btn-sm btn-icon" @click.stop="moveItemUp(checklist, itemIdx)" title="Move up">
                      <i class="fas fa-arrow-up fa-xs"></i>
                    </button>
                    <button class="btn btn-sm btn-icon" @click.stop="moveItemDown(checklist, itemIdx)"
                      title="Move down">
                      <i class="fas fa-arrow-down fa-xs"></i>
                    </button>
                    <button class="btn btn-sm btn-icon" @click.stop="deleteItem(checklist, item.id)"
                      title="Delete item">
                      <i class="fas fa-times"></i>
                    </button>
                  </div>
                </div>
              </div>

              <!-- Add item to this checklist -->
              <div class="add-item-row">
                <input v-model="newItemNames[checklist.id]" :id="'new-item-' + checklist.id" type="text"
                  class="add-item-input" placeholder="Add an item..." @keyup.enter="addItem(checklist)" />
                <button class="btn btn-sm btn-primary" @click="addItem(checklist)"
                  :disabled="!newItemNames[checklist.id]?.trim()">
                  <i class="fas fa-plus fa-sm"></i> Add
                </button>
              </div>
            </div>

            <!-- Add a new checklist -->
            <div class="add-checklist-row">
              <input v-model="newChecklistName" type="text" class="add-checklist-input" placeholder="Add checklist..."
                @keyup.enter="addChecklist" />
              <button class="btn btn-sm btn-primary" @click="addChecklist" :disabled="!newChecklistName.trim()">
                <i class="fas fa-plus fa-sm"></i> Add
              </button>
            </div>
          </div>

        </div>
      </div>
    </div>
  </div>
</template>

<script>
import VueTagsInput from '@voerro/vue-tagsinput'
import '@voerro/vue-tagsinput/dist/style.css'
import LabelBar from './LabelBar.vue';
import { MdEditor } from 'md-editor-v3'
import 'md-editor-v3/lib/style.css'

export default {
  name: 'Viewer',
  components: {
    VueTagsInput, LabelBar,MdEditor 
  },
  data() {
    return {
      main: null,
      isInitialized: false,
      default_list_name: "list",
      activeSettingsPanel: null,
      renamingIndex: null,
      renameValue: '',

      isDragging: false,
      startX: 0,
      scrollLeft: 0,

      // List drag and drop
      draggedListIndex: null,

      // Card drag and drop
      draggedCard: null,
      draggedCardSourceList: null,
      draggedCardIndex: null,
      draggedOverListId: null,
      dragOverTarget: null,

      // Add card functionality
      addingCardToListId: null,
      newCardName: '',
      cardIdCounter: Date.now(),

      // Add list functionality
      isAddingList: false,
      newListName: '',
      listIdCounter: Date.now(),

      // Context menu
      activeCardMenuId: null,
      cardMenuExpand: null,
      cardDataforMenu: null,

      // Label popup state
      showLabelPopup: false,
      labelPopupCard: null,
      newLabelName: '',
      selectedLabelColor: null,

      // Checklist editing state
      editingItemId: null,
      editItemName: '',
      newChecklistName: '',
      newItemNames: {},

      showPopup: false,
      selectedCard: {}
    };
  },
  props: {
    darkMode: Boolean,
    is_archived: {
      default: false,
      type: Boolean,
      required: true
    },
    labelNameCheckbox: {
      require: true,
      type: Boolean,
      default: false,
    },
    export_data: {
      default: null,
      type: String,
      required: true
    },
    labelColorMap: {
      required: true
    }
  },
  watch: {
    export_data(newVal) {
      if (newVal && newVal.length > 1) {
        this.main = JSON.parse(newVal);
        if (this.main.actions) {
          this.main.actions = []
        }
      }
    }
  },
  mounted() {
    document.addEventListener('click', this.closeCardMenuOnOutside);
    document.addEventListener('click', this.closeSettingsPanel);
    document.addEventListener('click', this.closeRenameOnClickOutside);
  },
  beforeUnmount() {
    document.removeEventListener('click', this.closeCardMenuOnOutside);
    document.removeEventListener('click', this.closeSettingsPanel);
    document.removeEventListener('click', this.closeRenameOnClickOutside);
  },
  created() {
    if (!this.isInitialized) {
      this.main = this.initMainData()
      this.isInitialized = true
    }
  },
  methods: {
    formatDate(dateString) {
      if (!dateString) return 'Not available';

      const date = new Date(dateString);
      const now = new Date();
      const diffTime = Math.abs(now - date);
      const diffDays = Math.floor(diffTime / (1000 * 60 * 60 * 24));

      // If less than 1 day ago, show relative time
      if (diffDays === 0) {
        const diffHours = Math.floor(diffTime / (1000 * 60 * 60));
        if (diffHours === 0) {
          const diffMinutes = Math.floor(diffTime / (1000 * 60));
          if (diffMinutes === 0) return 'Just now';
          return `${diffMinutes} minute${diffMinutes !== 1 ? 's' : ''} ago`;
        }
        return `${diffHours} hour${diffHours !== 1 ? 's' : ''} ago`;
      }

      // If less than 7 days ago, show days
      if (diffDays < 7) {
        return `${diffDays} day${diffDays !== 1 ? 's' : ''} ago`;
      }

      // Otherwise show full date
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'short',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      });
    },



    // ===== ADD THIS METHOD to handle card closing from anywhere =====
    closeCard(card) {
      if (!card) return;

      card.closed = true;
      card.dateClosed = new Date().toISOString();
      card.dateLastActivity = new Date().toISOString();

      this.main.cards = [...this.main.cards];
      this.$emit('card-archived', { card });
    },

    /**
      * Check if card has any indicators to show
      */
    hasCardIndicators(card) {
      return (card.dueDate) ||
        (card.desc && card.desc.trim().length > 0) ||
        this.hasChecklists(card);
    },

    /**
     * Check if a card has any checklists with items
     */
    hasChecklists(card) {
      if (!card.idChecklists || card.idChecklists.length === 0) return false;
      if (!this.main.checklists) return false;
      return this.main.checklists.some(cl =>
        card.idChecklists.includes(cl.id) && cl.checkItems && cl.checkItems.length > 0
      );
    },
    initMainData() {
      let Dtval = {}
      if (this.export_data && this.export_data.length > 1) {
        Dtval = JSON.parse(this.export_data)
      }
      Dtval.actions = []
      return Dtval;
    },

    // ===== Position Calculation Methods =====

    /**
     * Calculate position between two cards
     * Mimics Trello's floating-point position system
     */
    calculatePositionBetween(posA, posB) {
      if (!posA && !posB) return 65535;
      if (!posA) return posB / 2;
      if (!posB) return posA + 65535;
      return (posA + posB) / 2;
    },

    /**
     * Calculate position for a card at a specific index in a list
     */
    calculateCardPosition(listId, targetIndex) {
      const cards = this.getCardsByList(listId);

      if (cards.length === 0) {
        return 65535; // Default starting position
      }

      if (targetIndex === 0) {
        // Insert before first card
        return cards[0].pos / 2;
      }

      if (targetIndex >= cards.length) {
        // Insert after last card
        return cards[cards.length - 1].pos + 65535;
      }

      // Insert between two cards
      const prevCard = cards[targetIndex - 1];
      const nextCard = cards[targetIndex];
      return this.calculatePositionBetween(prevCard.pos, nextCard.pos);
    },

    /**
     * Get the target index based on drag position
     */
    getTargetIndex(cards, targetCardId, position) {
      if (!targetCardId) {
        return position === 'before' ? 0 : cards.length;
      }

      const targetIndex = cards.findIndex(c => c.id === targetCardId);
      if (targetIndex === -1) return cards.length;

      return position === 'before' ? targetIndex : targetIndex + 1;
    },

    /**
     * Normalize positions in a list if they get too close
     */
    normalizeListPositions(listId) {
      const cards = this.getCardsByList(listId);
      cards.forEach((card, index) => {
        card.pos = (index + 1) * 65535;
      });
      this.main.cards = [...this.main.cards];
    },

    /**
     * Check if normalization is needed
     */
    normalizeListPositionsIfNeeded(listId) {
      const cards = this.getCardsByList(listId);
      if (cards.length < 2) return;

      // Check if any adjacent cards have too close positions
      for (let i = 1; i < cards.length; i++) {
        const diff = cards[i].pos - cards[i - 1].pos;
        if (diff < 0.01) { // Threshold for "too close"
          this.normalizeListPositions(listId);
          return;
        }
      }
    },

    // ===== Checklist Methods =====

    getCompletedCount(checklist) {
      return checklist.checkItems.filter(item => item.state === 'complete').length;
    },

    addChecklist() {
      const name = this.newChecklistName.trim();
      if (!name) return;

      const checklistId = this.generateChecklistId();
      const card = this.selectedCard;

      const newChecklist = {
        id: checklistId,
        name: name,
        idBoard: this.main.lists.find(l => l.id === card.idList)?.idBoard || '',
        idCard: card.id,
        checkItems: [],
        pos: this.main.checklists.length * 65535
      };

      if (!this.main.checklists) {
        this.main.checklists = [];
      }
      this.main.checklists.push(newChecklist);

      if (!card.idChecklists) card.idChecklists = [];
      card.idChecklists.push(checklistId);

      this.newChecklistName = '';
      this.main.checklists = [...this.main.checklists];
      this.main.cards = [...this.main.cards];
    },

    deleteChecklist(checklistId) {
      const card = this.selectedCard;
      if (!card) return;

      card.idChecklists = card.idChecklists.filter(id => id !== checklistId);
      this.main.checklists = this.main.checklists.filter(cl => cl.id !== checklistId);

      this.main.cards = [...this.main.cards];
      this.main.checklists = [...this.main.checklists];
    },

    addItem(checklist) {
      const text = (this.newItemNames[checklist.id] || '').trim();
      if (!text) return;

      const itemId = this.generateCheckItemId();
      const newItem = {
        id: itemId,
        name: text,
        state: 'incomplete',
        pos: checklist.checkItems.length * 65535
      };

      checklist.checkItems.push(newItem);
      this.main.checklists = [...this.main.checklists];
      this.newItemNames[checklist.id] = '';
      this.$set(this.newItemNames, checklist.id, '');
    },

    deleteItem(checklist, itemId) {
      checklist.checkItems = checklist.checkItems.filter(item => item.id !== itemId);
      this.main.checklists = [...this.main.checklists];
    },

    moveItemUp(checklist, currentIdx) {
      if (currentIdx <= 0) return;
      const items = checklist.checkItems;
      [items[currentIdx], items[currentIdx - 1]] = [items[currentIdx - 1], items[currentIdx]];
      this.main.checklists = [...this.main.checklists];
    },

    moveItemDown(checklist, currentIdx) {
      if (currentIdx >= checklist.checkItems.length - 1) return;
      const items = checklist.checkItems;
      [items[currentIdx], items[currentIdx + 1]] = [items[currentIdx + 1], items[currentIdx]];
      this.main.checklists = [...this.main.checklists];
    },

    toggleItemState(checklist, item) {
      item.state = item.state === 'complete' ? 'incomplete' : 'complete';
      this.main.checklists = [...this.main.checklists];
    },

    startEditItem(item) {
      this.editingItemId = item.id;
      this.editItemName = item.name;
      this.$nextTick(() => {
        const input = this.$refs.itemEditInput;
        if (input) {
          if (Array.isArray(input)) input[0].focus();
          else input.focus();
        }
      });
    },

    saveEditItem(checklist, item) {
      if (this.editingItemId !== item.id) return;
      const newName = this.editItemName.trim();
      if (newName && newName !== item.name) {
        item.name = newName;
        this.main.checklists = [...this.main.checklists];
      }
      this.editingItemId = null;
      this.editItemName = '';
    },

    cancelEditItem() {
      this.editingItemId = null;
      this.editItemName = '';
    },

    // ===== ID Generators =====

    generateChecklistId() {
      let newId;
      do {
        const ts = Math.floor(Date.now() / 1000).toString(16).padStart(8, '0');
        const r1 = Math.random().toString(16).substring(2, 10);
        const r2 = Math.random().toString(16).substring(2, 10);
        newId = 'cl' + ts + r1 + r2;
      } while (this.main.checklists?.some(cl => cl.id === newId));
      return newId;
    },

    generateCheckItemId() {
      let newId;
      do {
        const ts = Math.floor(Date.now() / 1000).toString(16).padStart(8, '0');
        const r1 = Math.random().toString(16).substring(2, 10);
        newId = 'ci' + ts + r1;
      } while (this.main.checklists?.some(cl => cl.checkItems?.some(i => i.id === newId)));
      return newId;
    },

    getCardChecklists(checklistIds) {
      if (!checklistIds || checklistIds.length === 0 || !this.main.checklists) return [];
      return this.main.checklists.filter(checklist =>
        checklistIds.includes(checklist.id)
      );
    },

    // ===== Card Menu Methods =====

    toggleCardMenu(event = null, cardId = null, index = null, card = null) {
      this.activeCardMenuId = this.activeCardMenuId === cardId ? null : cardId;
      this.cardMenuExpand = this.cardMenuExpand === index ? null : index;
      this.cardDataforMenu = this.cardDataforMenu === card ? null : card;
    },

    closeCardMenuOnOutside(event) {
      if (!event.target.closest('.card-menu-btn') && !event.target.closest('.card-dropdown')) {
        this.toggleCardMenu(null, null, null, null);
      }
    },

    archiveCardFromMenu(card) {
      if (confirm(`Archive "${card.name}"?`)) {
        card.closed = true;
        this.main.cards = [...this.main.cards];
        this.activeCardMenuId = null;
        this.cardMenuExpand = null;
        this.cardDataforMenu = null;
      }
    },

    getCardIndexInList(card) {
      const cards = this.getCardsByList(card.idList);
      return cards.findIndex(c => c.id === card.id);
    },

    moveCardUp(card) {
      const cards = this.getCardsByList(card.idList);
      const currentIndex = cards.findIndex(c => c.id === card.id);
      if (currentIndex > 0) {
        this.reorderCardInSameList(card, currentIndex - 1);
      }
      this.activeCardMenuId = null;
      this.cardMenuExpand = null;
      this.cardDataforMenu = null;
    },

    moveCardDown(card) {
      const cards = this.getCardsByList(card.idList);
      const currentIndex = cards.findIndex(c => c.id === card.id);
      if (currentIndex < cards.length - 1) {
        this.reorderCardInSameList(card, currentIndex + 1);
      }
      this.activeCardMenuId = null;
      this.cardMenuExpand = null;
      this.cardDataforMenu = null;
    },

    moveCardTop(card) {
      const cards = this.getCardsByList(card.idList);
      const currentIndex = cards.findIndex(c => c.id === card.id);
      if (currentIndex > 0) {
        this.reorderCardInSameList(card, 0);
      }
      this.activeCardMenuId = null;
      this.cardMenuExpand = null;
      this.cardDataforMenu = null;
    },

    moveCardBottom(card) {
      const cards = this.getCardsByList(card.idList);
      const currentIndex = cards.findIndex(c => c.id === card.id);
      if (currentIndex < cards.length - 1) {
        this.reorderCardInSameList(card, cards.length - 1);
      }
      this.activeCardMenuId = null;
      this.cardMenuExpand = null;
      this.cardDataforMenu = null;
    },

    // ===== Label Popup Methods =====

    openLabelPopup(card) {
      this.labelPopupCard = card;
      this.showLabelPopup = true;
      this.newLabelName = '';
      this.selectedLabelColor = null;
      this.activeCardMenuId = null;
      this.cardMenuExpand = null;
      this.cardDataforMenu = null;
    },

    closeLabelPopup() {
      this.showLabelPopup = false;
      this.labelPopupCard = null;
    },

    addLabelToCard() {
      if (!this.selectedLabelColor) return;

      let name = this.newLabelName.trim();

      const card = this.labelPopupCard;
      const list = this.main.lists.find(l => l.id === card.idList);
      const boardId = list ? list.idBoard : '';
      const orgId = list ? list.idOrganization : '';

      const labelId = this.generateLabelId();
      const newLabel = {
        id: labelId,
        idBoard: boardId,
        idOrganization: orgId,
        name: name,
        color: this.selectedLabelColor
      };

      if (!card.labels) card.labels = [];
      card.labels.push(newLabel);

      if (!card.idLabels) card.idLabels = [];
      card.idLabels.push(labelId);

      this.newLabelName = '';
      this.selectedLabelColor = null;
      this.main.cards = [...this.main.cards];
    },

    removeLabelFromCard(labelId) {
      const card = this.labelPopupCard;
      if (!card) return;

      card.labels = card.labels.filter(l => l.id !== labelId);
      card.idLabels = card.idLabels.filter(id => id !== labelId);

      this.main.cards = [...this.main.cards];
    },

    generateLabelId() {
      let newId;
      do {
        const timestamp = Math.floor(Date.now() / 1000).toString(16).padStart(8, '0');
        const random1 = Math.random().toString(16).substring(2, 10);
        const random2 = Math.random().toString(16).substring(2, 10);
        newId = 'L' + timestamp + random1 + random2;
      } while (this.main.cards?.some(c => c.labels?.some(l => l.id === newId)));
      return newId;
    },

    handleTagsChanged(newTag, selectedCard) {
      if (selectedCard.tags) {
        let tagExists = selectedCard.tags.some(tag => tag.value === newTag.value);
        if (!tagExists) {
          selectedCard.tags.push({ "key": '', "value": newTag.value });
        }
      } else {
        selectedCard.tags = [];
        selectedCard.tags.push({ "key": '', "value": newTag.value });
      }

      if (this.main.exiting_tags) {
        let tagExists = this.main.exiting_tags.some(tag => tag.value === newTag.value);
        if (!tagExists) {
          this.main.exiting_tags.push({ "key": '', "value": newTag.value })
        }
      } else {
        this.main.exiting_tags = [];
        this.main.exiting_tags.push({ "key": '', "value": newTag.value })
      }
    },

    // ===== List Settings Methods =====

    toggleSettingsPanel(index) {
      if (this.activeSettingsPanel === index) {
        this.activeSettingsPanel = null;
      } else {
        this.activeSettingsPanel = index;
      }
    },

    startRename(index, event) {
      event.stopPropagation();
      this.renamingIndex = index;
      this.renameValue = this.main.lists[index].name;
      this.activeSettingsPanel = null;

      this.$nextTick(() => {
        const input = document.getElementById('rename-input-' + index);
        if (input) input.focus();
      });
    },

    saveRename(index) {
      if (this.renamingIndex === index) {
        if (this.renameValue && this.renameValue.trim()) {
          this.main.lists[index].name = this.renameValue.trim();
        }
        this.renamingIndex = null;
        this.renameValue = '';
      }
    },

    cancelRename() {
      this.renamingIndex = null;
      this.renameValue = '';
    },

    closeRenameOnClickOutside(e) {
      if (!e.target.closest('.rename-input') != null) {
        this.cancelRename();
      }
    },

    addListAfter(index) {
      this.newListName = this.default_list_name;
      this.addList(index + 1);
    },

    addListBefore(index) {
      this.newListName = this.default_list_name;
      this.addList(index);
    },

    moveListUp(index) {
      const lists = this.main.lists;
      const currentList = lists[index];

      let prevIndex = index - 1;
      while (prevIndex >= 0 && lists[prevIndex].closed) {
        prevIndex--;
      }

      if (prevIndex < 0) return;

      const temp = lists[index];
      lists[index] = lists[prevIndex];
      lists[prevIndex] = temp;

      lists.forEach((list, i) => {
        if (!list.closed) list.pos = i * 65535;
      });

      this.main.lists = [...lists];
      this.closePanel();
    },

    moveListDown(index) {
      const lists = this.main.lists;
      const currentList = lists[index];

      let nextIndex = index + 1;
      while (nextIndex < lists.length && lists[nextIndex].closed) {
        nextIndex++;
      }

      if (nextIndex >= lists.length) return;

      const temp = lists[index];
      lists[index] = lists[nextIndex];
      lists[nextIndex] = temp;

      lists.forEach((list, i) => {
        if (!list.closed) list.pos = i * 65535;
      });

      this.main.lists = [...lists];
      this.closePanel();
    },

    removeColor(index) {
      this.main.lists[index].color = null;
    },

    duplicateList(index) {
      const duplicated = { ...this.main.lists[index], name: `${this.main.lists[index].name} (Copy)` };
      this.main.lists.splice(index + 1, 0, duplicated);
      this.activeSettingsPanel = null;
    },

    archiveList(index) {
      if (confirm(`Archive "${this.main.lists[index].name}"?`)) {
        this.main.lists[index].closed = true;
      }
      this.activeSettingsPanel = null;
    },

    deleteList(index) {
      if (confirm(`Delete "${this.main.lists[index].name}" permanently?`)) {
        this.main.lists.splice(index, 1);
      }
      this.activeSettingsPanel = null;
    },

    closeSettingsPanel(e) {
      if (!e.target.closest('.header-dots')) {
        this.closePanel();
      }
    },

    closePanel() {
      this.activeSettingsPanel = null;
    },

    // ===== Card and List Getters =====

    getCardCount(listId) {
      if (!this.main.cards) return 0;
      return this.main.cards.filter(c => c.idList === listId && !c.closed).length;
    },

    getCardsByList(listId) {
      if (!this.main.cards) return [];
      return this.main.cards.filter(c => c.idList === listId && !c.closed).sort((a, b) => a.pos - b.pos);
    },

    // ===== Context Menu Methods =====

    showContextMenu(event, card) {
      event.preventDefault();
      this.showContextMenuFlag = true;
      this.contextMenuX = event.clientX;
      this.contextMenuY = event.clientY;
      this.contextMenuCard = card;
    },

    closeContextMenu() {
      this.showContextMenuFlag = false;
      this.contextMenuCard = null;
    },

    archiveCard() {
      if (this.contextMenuCard) {
        const card = this.main.cards.find(c => c.id === this.contextMenuCard.id);
        if (card) {
          card.closed = true;
          this.main.cards = [...this.main.cards];
          this.$emit('card-archived', { card: this.contextMenuCard });
        }
      }
      this.closeContextMenu();
    },

    // ===== ID Generators =====

    generateCardId() {
      let newId;
      do {
        const timestamp = Math.floor(Date.now() / 1000).toString(16).padStart(8, '0');
        const random1 = Math.random().toString(16).substring(2, 10);
        const random2 = Math.random().toString(16).substring(2, 10);
        newId = timestamp + random1 + random2;
      } while (this.main.cards && this.main.cards.some(card => card.id === newId));

      return newId;
    },

    generateTagId() {
      const timestamp = Math.floor(Date.now() / 1000).toString(16).padStart(8, '0');
      const random1 = Math.random().toString(16).substring(2, 10);
      const random2 = Math.random().toString(16).substring(2, 10);
      return 'T' + timestamp + random1 + random2;
    },

    generateListId() {
      let newId;
      do {
        const timestamp = Math.floor(Date.now() / 1000).toString(16).padStart(8, '0');
        const random1 = Math.random().toString(16).substring(2, 10);
        const random2 = Math.random().toString(16).substring(2, 10);
        newId = timestamp + random1 + random2;
      } while (this.main.lists && this.main.lists.some(list => list.id === newId));

      return newId;
    },

    generateShortId() {
      if (!this.main.cards) return 1;
      const maxShortId = Math.max(...this.main.cards.map(c => c.idShort || 0), 0);
      return maxShortId + 1;
    },

    generateShortLink() {
      const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
      let result = '';
      for (let i = 0; i < 8; i++) {
        result += chars.charAt(Math.floor(Math.random() * chars.length));
      }
      return result;
    },

    // ===== Add Card Methods =====

    showAddCardInput(listId) {
      this.addingCardToListId = listId;
      this.newCardName = '';

      this.$nextTick(() => {
        const inputs = this.$refs.cardInput;
        if (inputs) {
          const input = Array.isArray(inputs) ? inputs[inputs.length - 1] : inputs;
          if (input) input.focus();
        }
      });
    },

    addCard(listId) {
      const cardName = this.newCardName.trim();
      if (!cardName) return;

      const cardId = this.generateCardId();
      const shortId = this.generateShortId();
      const now = new Date().toISOString();

      // Calculate position using the new system
      const cardsInList = this.getCardsByList(listId);
      const pos = this.calculateCardPosition(listId, cardsInList.length);

      const list = this.main.lists.find(l => l.id === listId);

      const newCard = {
        id: cardId,
        address: null,
        agent: {
          name: null,
          conversationId: null
        },
        badges: {
          fogbugz: "",
          checkItems: 0,
          checkItemsChecked: 0,
          checkItemsEarliestDue: null,
          comments: 0,
          attachments: 0,
          description: false,
          due: null,
          dueComplete: true,
          start: null,
          lastUpdatedByAi: false,
          attachmentsByType: {
            trello: {
              board: 0,
              card: 0
            }
          },
          externalSource: null,
          location: false,
          votes: 0,
          maliciousAttachments: 0,
          viewingMemberVoted: false,
          subscribed: false
        },
        tags: [],
        checkItemStates: [],
        closed: false,
        coordinates: null,
        creationMethod: null,
        creationMethodError: null,
        creationMethodLoadingStartedAt: null,
        dueComplete: true,
        dateClosed: null,
        createdAt: now,
        dateLastActivity: now,
        dateCompleted: now,
        dateViewedByCreator: null,
        desc: "",
        descData: {
          emoji: {}
        },
        due: null,
        dueReminder: null,
        email: `card-${cardId}@boards.trello.com`,
        externalSource: null,
        faviconUrl: null,
        idBoard: list?.idBoard || "",
        idChecklists: [],
        idLabels: [],
        idList: listId,
        idMemberCreator: null,
        idMembers: [],
        idMembersVoted: [],
        idOrganization: list?.idOrganization || "",
        idShort: shortId,
        idAttachmentCover: null,
        labels: [],
        limits: {
          attachments: {
            perCard: {
              status: "ok",
              disableAt: 1000,
              warnAt: 800
            }
          },
          checklists: {
            perCard: {
              status: "ok",
              disableAt: 500,
              warnAt: 400
            }
          },
          stickers: {
            perCard: {
              status: "ok",
              disableAt: 70,
              warnAt: 56
            }
          }
        },
        locationName: null,
        manualCoverAttachment: false,
        name: cardName,
        nodeId: `ari:cloud:trello::card/workspace/${list?.idOrganization || ""}/${cardId}`,
        originalDesc: "",
        originalName: "",
        pinned: false,
        pos: pos,
        recurrenceRule: null,
        shortLink: this.generateShortLink(),
        shortUrl: `https://trello.com/c/${this.generateShortLink()}`,
        singleInstrumentationId: null,
        sourceEmail: null,
        staticMapUrl: null,
        start: null,
        subscribed: false,
        url: `https://trello.com/c/${this.generateShortLink()}/${shortId}-${cardName.toLowerCase().replace(/\s+/g, '-')}`,
        urlSource: null,
        urlSourceText: null,
        cover: {
          idAttachment: null,
          color: null,
          idUploadedBackground: null,
          size: "normal",
          brightness: "dark",
          yPosition: 0.5,
          idPlugin: null
        },
        isTemplate: false,
        cardRole: null,
        mirrorSourceId: null,
        mirrorSourceNodeId: null,
        attachments: [],
        pluginData: [],
        customFieldItems: []
      };

      if (!this.main.cards) {
        this.main.cards = [];
      }
      this.main.cards.push(newCard);
      this.main.cards = [...this.main.cards];

      this.addingCardToListId = null;
      this.newCardName = '';

      this.$emit('card-added', { card: newCard, listId });
    },

    cancelAddCard() {
      this.addingCardToListId = null;
      this.newCardName = '';
    },

    // ===== Add List Methods =====

    showAddListInput() {
      this.isAddingList = true;
      this.newListName = '';

      this.$nextTick(() => {
        const inputs = this.$refs.listInput;
        if (inputs) {
          const input = Array.isArray(inputs) ? inputs[inputs.length - 1] : inputs;
          if (input) input.focus();
        }
      });
    },

    addList(insertIndex = null) {
      const listName = this.newListName.trim();
      if (!listName) return;

      const listId = this.generateListId();
      const pos = this.calculateNewListPos();

      const existingList = this.main.lists && this.main.lists.length > 0 ? this.main.lists[0] : null;

      const newList = {
        id: listId,
        name: listName,
        closed: false,
        color: null,
        idBoard: existingList?.idBoard || "",
        pos: pos,
        subscribed: false,
        softLimit: null,
        type: null,
        datasource: {
          filter: false
        },
        creationMethod: null,
        idOrganization: existingList?.idOrganization || "",
        limits: {
          cards: {
            openPerList: {
              status: "ok",
              disableAt: 500,
              warnAt: 400
            },
            totalPerList: {
              status: "ok",
              disableAt: 1000,
              warnAt: 800
            }
          }
        },
        nodeId: `/workspace/${existingList?.idOrganization || ""}/${listId}`
      };

      if (!this.main.lists) {
        this.main.lists = [];
      }


      if (insertIndex !== null) {
        const clampedIndex = Math.max(0, Math.min(insertIndex, this.main.lists.length));
        this.main.lists.splice(clampedIndex, 0, newList);

        // Recalculate positions for all lists
        this.main.lists.forEach((list, i) => {
          if (!list.closed) list.pos = (i + 1) * 65535;
        });
      } else {
        this.addList(this.main.lists.length + 1)
      }

      // Recalculate positions for all non-closed lists
      const openLists = this.main.lists.filter(l => !l.closed);
      openLists.forEach((list, i) => {
        list.pos = (i + 1) * 65535;
      });

      // Trigger reactivity
      this.main.lists = [...this.main.lists];

      this.isAddingList = false;
      this.newListName = '';

      this.$emit('list-added', { list: newList });
    },

    cancelAddList() {
      this.isAddingList = false;
      this.newListName = '';
    },

    calculateNewListPos() {
      if (!this.main.lists || this.main.lists.length === 0) return 65535;
      const maxPos = Math.max(...this.main.lists.map(l => l.pos || 0));
      return maxPos + 65535;
    },

    // ===== List Drag and Drop Methods =====

    onListDragStart(event, index) {
      if (!event.target.closest('.header')) {
        event.preventDefault();
        return false;
      }

      this.draggedListIndex = index;
      event.dataTransfer.effectAllowed = 'move';
      event.dataTransfer.setData('text/plain', `list-${index}`);

      const dragElement = event.target.closest('.list-item');
      if (dragElement) {
        const clone = dragElement.cloneNode(true);
        clone.style.position = 'absolute';
        clone.style.top = '-1000px';
        clone.style.left = '-1000px';
        clone.style.width = dragElement.offsetWidth + 'px';
        clone.style.opacity = '0.99';
        clone.style.transform = 'rotate(20deg)';
        clone.style.boxShadow = '0 10px 30px rgba(0,0,0,0.3)';
        document.body.appendChild(clone);
        event.dataTransfer.setDragImage(clone, 20, 20);
        setTimeout(() => document.body.removeChild(clone), 0);
      }

      return true;
    },

    onListDragEnd() {
      this.draggedListIndex = null;
    },

    onListDragOver(event, index) {
      event.preventDefault();
      event.dataTransfer.dropEffect = 'move';

      if (this.draggedListIndex === null) return;
      if (this.draggedListIndex === index) return;

      const draggedList = this.main.lists[this.draggedListIndex];
      const newLists = [...this.main.lists];

      newLists.splice(this.draggedListIndex, 1);

      let newIndex = index;
      if (this.draggedListIndex < index) {
        newIndex = index - 1;
      }

      newLists.splice(newIndex, 0, draggedList);
      this.main.lists = newLists;
      this.draggedListIndex = newIndex;
    },

    onListDrop(event, index) {
      event.preventDefault();
      if (this.draggedListIndex !== null) {
        // Recalculate positions
        this.main.lists.forEach((list, i) => {
          if (!list.closed) list.pos = (i + 1) * 65535;
        });
        this.main.lists = [...this.main.lists];
        this.$emit('lists-reordered', this.main.lists);
      }
      this.draggedListIndex = null;
    },

    // ===== Card Drag and Drop Methods (FIXED) =====

    onCardDragStart(event, card, cardIndex, sourceListId) {
      event.stopPropagation();
      this.draggedCard = card;
      this.draggedCardSourceList = sourceListId;
      this.draggedCardIndex = cardIndex;

      event.dataTransfer.effectAllowed = 'move';
      event.dataTransfer.setData('text/plain', JSON.stringify({
        cardId: card.id,
        sourceList: sourceListId,
        cardIndex: cardIndex
      }));

      // Add dragging class
      const cardElement = event.target.closest('.card');
      if (cardElement) {
        cardElement.classList.add('card-dragging');
      }

      return true;
    },

    onCardDragEnd() {
      // Clean up all drag indicators
      document.querySelectorAll('.card').forEach(el => {
        el.classList.remove('drag-over-card', 'drag-over-card-after', 'card-dragging');
      });

      this.draggedCard = null;
      this.draggedCardSourceList = null;
      this.draggedCardIndex = null;
      this.draggedOverListId = null;
      this.dragOverTarget = null;
    },

    onCardContentDragOver(event, listId) {
      event.preventDefault();
      event.dataTransfer.dropEffect = 'move';

      if (!this.draggedCard) return;

      // Show that this list can accept the card
      if (this.draggedCardSourceList !== listId) {
        this.draggedOverListId = listId;
      }

      // Don't modify card positions during drag over empty area
      this.dragOverTarget = null;
    },

    onCardContentDragLeave(event, listId) {
      const contentElement = event.currentTarget;
      if (!contentElement.contains(event.relatedTarget)) {
        if (this.draggedOverListId === listId) {
          this.draggedOverListId = null;
        }
        this.dragOverTarget = null;
      }
    },

    onCardDragOverCard(event, targetCard, targetIndex, targetListId) {
      event.preventDefault();
      event.stopPropagation();
      event.dataTransfer.dropEffect = 'move';

      if (!this.draggedCard) return;
      if (this.draggedCard.id === targetCard.id) return;

      this.draggedOverListId = null;

      const targetElement = event.currentTarget;
      const rect = targetElement.getBoundingClientRect();
      const mouseY = event.clientY;
      const relativePosition = (mouseY - rect.top) / rect.height;

      const position = relativePosition <= 0.5 ? 'before' : 'after';

      this.dragOverTarget = {
        cardId: targetCard.id,
        position: position
      };

      // Update visual indicators only
      document.querySelectorAll('.card').forEach(el => {
        el.classList.remove('drag-over-card', 'drag-over-card-after');
      });

      const cardElement = event.currentTarget;
      if (position === 'before') {
        cardElement.classList.add('drag-over-card');
      } else {
        cardElement.classList.add('drag-over-card-after');
      }
    },

    onCardDragLeaveCard(event, targetCard, targetIndex, targetListId) {
      const cardElement = event.currentTarget;
      if (!cardElement.contains(event.relatedTarget)) {
        if (this.dragOverTarget && this.dragOverTarget.cardId === targetCard.id) {
          this.dragOverTarget = null;
          cardElement.classList.remove('drag-over-card', 'drag-over-card-after');
        }
      }
    },

    onCardDropOnCard(event, targetCard, targetIndex, targetListId) {
      event.preventDefault();
      event.stopPropagation();

      if (!this.draggedCard) return;

      const cards = this.getCardsByList(targetListId);
      const position = this.dragOverTarget?.position || 'after';
      const targetInsertIndex = this.getTargetIndex(cards, targetCard.id, position);

      if (this.draggedCardSourceList === targetListId) {
        // Same list - reorder
        this.reorderCardInSameList(this.draggedCard, targetInsertIndex);
      } else {
        // Different list - move
        this.moveCardToDifferentList(
          this.draggedCard,
          this.draggedCardSourceList,
          targetListId,
          targetInsertIndex
        );
      }

      // Clean up
      this.onCardDragEnd();
    },

    onCardDrop(event, targetListId) {
      event.preventDefault();

      if (!this.draggedCard) return;

      if (this.draggedCardSourceList !== targetListId) {
        const cards = this.getCardsByList(targetListId);
        this.moveCardToDifferentList(
          this.draggedCard,
          this.draggedCardSourceList,
          targetListId,
          cards.length // Add to end
        );
      }

      // Clean up
      this.onCardDragEnd();
    },

    // ===== Card Movement Methods (FIXED) =====

    reorderCardInSameList(card, targetIndex) {
      const cards = this.getCardsByList(card.idList);
      const currentIndex = cards.findIndex(c => c.id === card.id);

      if (currentIndex === targetIndex) return;
      if (currentIndex === -1) return;

      // Remove card from current position
      cards.splice(currentIndex, 1);

      // Insert at target position
      cards.splice(targetIndex, 0, card);

      // Recalculate all positions
      cards.forEach((c, idx) => {
        c.pos = (idx + 1) * 65535;
      });

      // Check if normalization is needed
      this.normalizeListPositionsIfNeeded(card.idList);

      this.main.cards = [...this.main.cards];
      this.$emit('card-reordered', { card, fromIndex: currentIndex, toIndex: targetIndex });
    },

    moveCardToList(card, sourceListId, targetListId) {
      this.moveCardToDifferentList(card, sourceListId, targetListId, this.getCardsByList(targetListId).length);
    },

    moveCardToDifferentList(card, sourceListId, targetListId, targetIndex) {
      const actualCard = this.main.cards.find(c => c.id === card.id);
      if (!actualCard) return;

      // Update the card's list ID
      actualCard.idList = targetListId;

      // Get cards in target list (excluding the moved card)
      const targetCards = this.getCardsByList(targetListId).filter(c => c.id !== card.id);

      // Insert at the correct position
      targetCards.splice(targetIndex, 0, actualCard);

      // Update positions for target list
      targetCards.forEach((c, idx) => {
        c.pos = (idx + 1) * 65535;
      });

      // Update positions for source list
      const sourceCards = this.getCardsByList(sourceListId);
      sourceCards.forEach((c, idx) => {
        c.pos = (idx + 1) * 65535;
      });

      // Normalize if needed
      this.normalizeListPositionsIfNeeded(targetListId);
      this.normalizeListPositionsIfNeeded(sourceListId);

      this.main.cards = [...this.main.cards];
      this.$emit('card-moved', { card, sourceListId, targetListId, targetIndex });
    },

    // ===== Drag Scrolling Methods =====

    startDrag(e) {
      if (e.target === this.$el || (this.$el.contains(e.target) && e.target === this.$el)) {
        this.isDragging = true;
        this.startX = e.pageX - this.$el.offsetLeft;
        this.scrollLeft = this.$el.scrollLeft;
      }
    },

    onDrag(e) {
      if (!this.isDragging) return;
      e.preventDefault();
      const x = e.pageX - this.$el.offsetLeft;
      const walk = (x - this.startX) * 2;
      this.$el.scrollLeft = this.scrollLeft - walk;
    },

    stopDrag(e) {
      this.isDragging = false;
    },

    // ===== Card Popup Methods =====

    openCardPopup(cardId) {
      const card = this.main.cards.find(c => c.id === cardId);
      if (card) {
        // Update last activity when card is opened
        card.dateLastActivity = new Date().toISOString();

        // If card is being closed through the popup, update dateClosed
        if (card.closed && !card.dateClosed) {
          card.dateClosed = new Date().toISOString();
        }

        this.selectedCard = card;
        this.showPopup = true;
        this.main.cards = [...this.main.cards]; // Trigger reactivity
      }
    },

    closePopup() {
      this.showPopup = false;
      this.selectedCard = {};
    },

    getListName(listId) {
      if (!this.main.lists) return 'Unknown list';
      const list = this.main.lists.find(l => l.id === listId);
      return list ? list.name : 'Unknown list';
    }
  },
  computed: {

  },
};
</script>