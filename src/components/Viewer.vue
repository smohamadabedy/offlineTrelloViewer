<template>
  
  <div class="scroll-container m-0 pt-5" @mousedown="startDrag" @mousemove="onDrag" @mouseup="stopDrag" @mouseleave="stopDrag" @contextmenu.prevent>
    
    <div id="zoom-canvas" class="scroll-row">

      <!-- List Section -->  
      <template v-for="(list, index) in main.lists" :key="index">
        <div class="list-item" v-if="list.closed==is_archived" :data-list-index="index" :data-list-id="list.id" :class="{ 'dragging-source': draggedListIndex === index,  'drag-over-list': draggedOverListId === list.id && draggedCardSourceList !== list.id}">
          <!-- Header -->
          <div class="header" draggable="true" @dragstart="onListDragStart($event, index)" @dragend="onListDragEnd" @dragover="onListDragOver($event, index)" @drop="onListDrop($event, index)">
            
            <div class="header-colorbar" v-if="list.color != null" :style="{'background-color': list.color}"></div>
            
            <div class="drag-handle">⋮⋮</div>
            
            <div class="list-title" v-if="renamingIndex !== index">
              <span @click="startRename(index,$event)" class="cursor-text">{{list.name}}</span>
            </div>
            <input v-else v-model="renameValue" @click.stop @change="saveRename(index)" @keyup.esc="cancelRename" @mousedown.stop class="rename-input" ref="renameInput" autofocus/>
            
            <div class="header-dots">
              <button class="btn btn-sm" @click.stop="toggleSettingsPanel(index)"><span class="d-block">⋮</span></button>
              <!-- Settings Panel -->
              <div v-if="activeSettingsPanel === index" class="settings-panel">
                <div class="settings-item">
                  <button @click="startRename(index,$event)"><small>✏️ Rename</small></button>
                </div>
                <div class="settings-item">
                  <div>
                    <input type="color" v-model="list.color" class="btn color-custom" title="Custom color"/>
                    <button @click="removeColor(index)"class="p-2"><small>✕</small></button>
                  </div>
                </div>
                <div class="settings-item settings-item-row">
                  <button @click="addListBefore(index)"><small>|← <br/>Add</small></button>
                  <button @click="addListAfter(index)"> <small>→| <br/>Add</small></button>
                </div>
                <div class="settings-item settings-item-row">
                  <button @click="moveListUp(index)"> <small>← <br/> Move</small></button>
                  <button @click="moveListDown(index)"><small>→ <br/> Move</small></button>
                </div>
                <div class="settings-item">
                  <button @click="duplicateList(index)"><small>📋 Duplicate</small></button>
                </div>
                <div class="settings-item">
                  <button @click="archiveList(index)"><small>📦 Archive</small></button>
                </div>
                <div class="settings-item">
                  <button @click="deleteList(index)" class="danger-btn"><small>🗑️ Delete</small></button>
                </div>
              </div>
            </div>

          </div>
          <!-- Content -->
          <div 
            class="content"
            @dragover="onCardContentDragOver($event, list.id)"
            @dragleave="onCardContentDragLeave($event, list.id)"
            @drop="onCardDrop($event, list.id)"
          >
            <div 
              v-for="(card, cardIndex) in getCardsByList(list.id)" 
              :key="card.id"
              class="card"
              :data-card-id="card.id"
              :data-card-index="cardIndex"
              :data-source-list="list.id"
              :class="{
                'card-dragging': draggedCard && draggedCard.id === card.id,
                'drag-over-card': dragOverTarget && dragOverTarget.cardId === card.id && dragOverTarget.position === 'before',
                'drag-over-card-after': dragOverTarget && dragOverTarget.cardId === card.id && dragOverTarget.position === 'after'
              }"
              draggable="true"
              @dragstart="onCardDragStart($event, card, cardIndex, list.id)"
              @dragend="onCardDragEnd"
              @dragover="onCardDragOverCard($event, card, cardIndex, list.id)"
              @dragleave="onCardDragLeaveCard($event, card, cardIndex, list.id)"
              @drop="onCardDropOnCard($event, card, cardIndex, list.id)"
              @click.stop="openCardPopup(card.id)"
              @contextmenu.stop="showContextMenu($event, card)"
            >
              <div class="card-title">{{ card.name }}</div>
            </div>
            <div 
              v-if="getCardsByList(list.id).length === 0"
              class="empty-list-hint"
            >
              Drop cards here
            </div>
           
          </div>
          <!-- Footer -->
          <div class="footer">
            <!-- Add Card Button -->
            <div  v-if="!addingCardToListId || addingCardToListId !== list.id"  class="add-card-button"  @click.stop="showAddCardInput(list.id)">+ Add a card</div>
            <!-- Add Card Input -->
            <div v-else class="add-card-form" @click.stop>
              <input ref="cardInput" v-model="newCardName" type="text" class="add-card-input" placeholder="Enter card title..."  @keyup.enter="addCard(list.id)"  @keyup.escape="cancelAddCard"/>
              <div class="add-card-actions">
                <button  class="add-card-submit"   @click="addCard(list.id)"  :disabled="!newCardName.trim()">
                  Add Card
                </button>
                <button class="add-card-cancel" @click="cancelAddCard">✕</button>
              </div>
            </div>
          </div>
          <!-- EOL -->
        </div>
      </template>
    
      <!-- Add New List Section -->
      <div class="list-item add-list-item">
        <div v-if="!isAddingList" class="add-list-button" @click="showAddListInput">+ Add another list</div>
        <div v-else class="add-list-form" @click.stop>
          <input ref="listInput" v-model="newListName" type="text" class="add-list-input" placeholder="Enter list title..." @keyup.enter="addList" @keyup.escape="cancelAddList"/>
          <div class="add-list-actions">
            <button class="add-list-submit" @click="addList" :disabled="!newListName.trim()">Add List</button>
            <button class="add-list-cancel" @click="cancelAddList">✕</button>
          </div>
        </div>
      </div>

      <div>&nbsp;</div>

    </div>
    
    <!-- Context Menu -->
    <div 
      v-if="showContextMenuFlag"
      class="context-menu"
      :style="{ top: contextMenuY + 'px', left: contextMenuX + 'px' }"
      @click.stop
    >
      <div class="context-menu-item" >
            <input 
              type="date" 
              v-model="this.main.cards.find(c => c.id === this.contextMenuCard.id).dueDate"
              class="date-picker-input"
              :class="{ 'no-date': !selectedCard.dueDate }"
              :data-placeholder="selectedCard.dueDate ? '' : 'No due date'"
            />
      </div>
      <div class="context-menu-item" @click="archiveCard">
        <span class="context-menu-icon">📦</span>
        Archive Card
      </div>

      <div class="context-menu-item" @click="archiveCard">
        <span class="context-menu-icon">📦</span>
        Move Up
      </div>

      <div class="context-menu-item" @click="archiveCard">
        <span class="context-menu-icon">📦</span>
        Move Down
      </div>

      <div class="context-menu-item" @click="archiveCard">
        <span class="context-menu-icon">📦</span>
        Move Top
      </div>


      <div class="context-menu-item" @click="archiveCard">
        <span class="context-menu-icon">📦</span>
        Move Bottom
      </div>


    </div>
    
    <!-- Popup -->
    <div v-if="showPopup" class="popup-overlay" @click="closePopup">
      <div class="popup-content" @click.stop>
        <!-- Fixed Header -->
        <div class="popup-header">
          <h2><span>{{ getListName(selectedCard.idList) }}</span> &#129154; {{ selectedCard.name }}</h2>
          <button class="close-btn" @click="closePopup">×</button>
        </div>
        
        <!-- Scrollable Body -->
        <div class="popup-body">
          <div class="popup-field">
            <label>Description:</label>
            <textarea v-model="selectedCard.desc" placeholder="Add a more detailed description..."></textarea>
          </div>

          <div class="popup-field">
            <label>Due Date:</label>
            <div class="row g-2 align-items-center">
              <div class="col-6 col-md-8">
                <p class="due-date-text mb-0">{{ selectedCard.dueDate || 'No due date' }}</p>
              </div>
              <div class="col-6 col-md-4">
                <input 
                  type="date" 
                  v-model="selectedCard.dueDate"
                  class="date-picker-input form-control"
                  :class="{ 'no-date': !selectedCard.dueDate }"
                  :data-placeholder="selectedCard.dueDate ? '' : 'No due date'"
                />
              </div>
            </div>
          </div>
          
          <div v-if="getCardChecklists(selectedCard.idChecklists).length > 0" class="popup-field">
            <label>Checklists:</label>
            <div 
              v-for="checklist in getCardChecklists(selectedCard.idChecklists)" 
              :key="checklist.id"
              class="checklist"
            >
              <div class="checklist-title"></div>
              <div class="checklist-items">
                <div 
                  v-for="item in checklist.checkItems" 
                  :key="item.id"
                  class="checklist-item"
                >
                  <input 
                    type="checkbox" 
                    :checked="item.state === 'complete'"
                    :id="item.id"
                    class="checklist-checkbox"
                  />
                  <label :for="item.id" class="checklist-label">{{ item.name }}</label>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

  </div>
</template>

<script>
export default {
  name: 'Viewer',
  data() {
    return {
      main: this.export_data && this.export_data.length > 1 ? JSON.parse(this.export_data) : { lists: [], cards: [], checklists: [] },
      default_list_name: "list",
      activeSettingsPanel: null, // Track which list's panel is open
      renamingIndex: null, // Track which list is being renamed
      renameValue: '', // Store the new name value
            
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
      showContextMenuFlag: false,
      contextMenuX: 0,
      contextMenuY: 0,
      contextMenuCard: null,
      
      showPopup: false,
      selectedCard: {}
    };
  },
  props: {
    is_archived:{
      default:false,
      type: Boolean,
      required: true
    },
    export_data: {
      default: null,
      type: String,
      required: true
    }
  },
  watch: {
    export_data(newVal) {
      if (newVal && newVal.length > 1) {
        this.main = JSON.parse(newVal);
      }
      this.$emit('update-expand', 'Updated from child!');
    }
  },
  mounted() {
    // Close context menu when clicking anywhere
    document.addEventListener('click', this.closeContextMenu);
    document.addEventListener('click', this.closeSettingsPanel);
    document.addEventListener('click', this.closeRenameOnClickOutside);

  },
  beforeUnmount() {
    document.removeEventListener('click', this.closeContextMenu);
    document.removeEventListener('click', this.closeSettingsPanel);
    document.removeEventListener('click', this.closeRenameOnClickOutside);
  },
  methods: {
    /* list settings */
    toggleSettingsPanel(index) {
      if (this.activeSettingsPanel === index) {
        this.activeSettingsPanel = null; // Close if already open
      } else {
        this.activeSettingsPanel = index; // Open this one
      }
    },
    
    /* Renaming */
    startRename(index,event) {
      event.stopPropagation(); // Prevent event from bubbling
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
      // Check if click is outside the rename input
      if ( !e.target.closest('.rename-input') != null  ) {
        this.cancelRename();
      }
    },
    /* RenamingEnd */

    addListAfter(index){
      this.newListName = this.default_list_name;
      this.addList(index+1)
    },
    addListBefore(index){
      this.newListName = this.default_list_name;
      this.addList(index-1)
    },
    moveListUp(index) {
      const lists = this.main.lists;
      const currentList = lists[index];
      
      // Find the previous non-closed list
      let prevIndex = index - 1;
      while (prevIndex >= 0 && lists[prevIndex].closed) {
        prevIndex--;
      }
      
      if (prevIndex < 0) return; // No non-closed list above
      
      // Swap
      const temp = lists[index];
      lists[index] = lists[prevIndex];
      lists[prevIndex] = temp;
      
      // Update positions
      lists.forEach((list, i) => {
        if (!list.closed) list.pos = i * 65535;
      });
      
      this.main.lists = [...lists];
      this.closePanel();
    },

    moveListDown(index) {
      const lists = this.main.lists;
      const currentList = lists[index];
      
      // Find the next non-closed list
      let nextIndex = index + 1;
      while (nextIndex < lists.length && lists[nextIndex].closed) {
        nextIndex++;
      }
      
      if (nextIndex >= lists.length) return; // No non-closed list below
      
      // Swap
      const temp = lists[index];
      lists[index] = lists[nextIndex];
      lists[nextIndex] = temp;
      
      // Update positions
      lists.forEach((list, i) => {
        if (!list.closed) list.pos = i * 65535;
      });
      
      this.main.lists = [...lists];
      this.closePanel();
    },  
    removeColor(index){
      this.main.lists[index].color = null
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
    // Close panel when clicking outside (add to mounted or created)
    closeSettingsPanel(e) {
      if (!e.target.closest('.header-dots')) {
        this.closePanel();
      }
    },
    closePanel(){
      this.activeSettingsPanel = null;
    },
    /* list settings */


    getCardCount(listId) {
      if (!this.main.cards) return 0;
      return this.main.cards.filter(c => c.idList === listId && !c.closed).length;
    },
    
    getCardsByList(listId) {
      if (!this.main.cards) return [];
      return this.main.cards.filter(c => c.idList === listId && !c.closed).sort((a, b) => a.pos - b.pos);
    },
    
    // Context Menu Methods
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
    
    // Generate unique ID for new cards
    generateCardId() {
      let newId;
      do {
        // 24 hex characters like Trello
        const timestamp = Math.floor(Date.now() / 1000).toString(16).padStart(8, '0');
        const random1 = Math.random().toString(16).substring(2, 10);
        const random2 = Math.random().toString(16).substring(2, 10);
        newId = timestamp + random1 + random2;
      } while (this.main.cards && this.main.cards.some(card => card.id === newId));
      
      return newId;
    },

    // Generate unique ID for new lists
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
    
    // Generate short ID
    generateShortId() {
      if (!this.main.cards) return 1;
      const maxShortId = Math.max(...this.main.cards.map(c => c.idShort || 0), 0);
      return maxShortId + 1;
    },
    
    // Calculate position for new card
    calculateNewCardPos(listId) {
      const cardsInList = this.main.cards.filter(c => c.idList === listId && !c.closed);
      if (cardsInList.length === 0) return 65535;
      const maxPos = Math.max(...cardsInList.map(c => c.pos || 0));
      return maxPos + 65535;
    },
    
    // Calculate position for new list
    calculateNewListPos() {
      if (!this.main.lists || this.main.lists.length === 0) return 65535;
      const maxPos = Math.max(...this.main.lists.map(l => l.pos || 0));
      return maxPos + 65535;
    },
    
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
      const pos = this.calculateNewCardPos(listId);
      
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
        checkItemStates: [],
        closed: false,
        coordinates: null,
        creationMethod: null,
        creationMethodError: null,
        creationMethodLoadingStartedAt: null,
        dueComplete: true,
        dateClosed: null,
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
    
    generateShortLink() {
      const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
      let result = '';
      for (let i = 0; i < 8; i++) {
        result += chars.charAt(Math.floor(Math.random() * chars.length));
      }
      return result;
    },
    
    cancelAddCard() {
      this.addingCardToListId = null;
      this.newCardName = '';
    },
    
    // Add List Methods
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
      
      // Get board and organization from existing lists
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
        // Clamp index between 0 and lists length
        const clampedIndex = Math.max(0, Math.min(insertIndex, this.main.lists.length));
        this.main.lists.splice(clampedIndex, 0, newList);
      } else {
        this.main.lists.push(newList);
      }
      
      this.main.lists = [...this.main.lists];
      
      this.isAddingList = false;
      this.newListName = '';
      
      this.$emit('list-added', { list: newList });
    },
    
    cancelAddList() {
      this.isAddingList = false;
      this.newListName = '';
    },
    
    // List drag and drop methods
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
        this.$emit('lists-reordered', this.main.lists);
      }
      this.draggedListIndex = null;
    },

    // Card drag and drop methods
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
      
      return true;
    },

    onCardDragEnd() {
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
      
      if (this.draggedCardSourceList !== listId) {
        this.draggedOverListId = listId;
      }
      
      const cards = this.getCardsByList(listId);
      if (cards.length === 0) {
        this.dragOverTarget = null;
      } else {
        const lastCard = cards[cards.length - 1];
        this.dragOverTarget = {
          cardId: lastCard.id,
          position: 'after'
        };
      }
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
      
      if (this.draggedCardSourceList === targetListId) {
        let newIndex = targetIndex;
        if (position === 'after') {
          newIndex = targetIndex + 1;
        }
        
        let adjustedNewIndex = newIndex;
        if (this.draggedCardIndex < newIndex) {
          adjustedNewIndex = newIndex - 1;
        }
        
        if (adjustedNewIndex !== this.draggedCardIndex) {
          const listCards = this.main.cards
            .filter(c => c.idList === targetListId && !c.closed)
            .sort((a, b) => a.pos - b.pos);
          
          const [movedCard] = listCards.splice(this.draggedCardIndex, 1);
          listCards.splice(adjustedNewIndex, 0, movedCard);
          
          listCards.forEach((card, idx) => {
            card.pos = idx;
          });
          
          this.main.cards = [...this.main.cards];
          this.draggedCardIndex = adjustedNewIndex;
        }
      }
    },

    onCardDragLeaveCard(event, targetCard, targetIndex, targetListId) {
      const cardElement = event.currentTarget;
      if (!cardElement.contains(event.relatedTarget)) {
        if (this.dragOverTarget && this.dragOverTarget.cardId === targetCard.id) {
          this.dragOverTarget = null;
        }
      }
    },

    onCardDropOnCard(event, targetCard, targetIndex, targetListId) {
      event.preventDefault();
      event.stopPropagation();
      
      if (!this.draggedCard) return;
      
      if (this.draggedCardSourceList === targetListId) {
        this.$emit('card-reordered', { 
          card: this.draggedCard, 
          fromIndex: this.draggedCardIndex, 
          toIndex: targetIndex 
        });
      } else {
        const position = this.dragOverTarget?.position || 'after';
        let insertIndex = targetIndex;
        if (position === 'after') {
          insertIndex = targetIndex + 1;
        }
        
        this.moveCardToDifferentList(this.draggedCard, this.draggedCardSourceList, targetListId, insertIndex);
      }
      
      this.draggedCard = null;
      this.draggedCardSourceList = null;
      this.draggedCardIndex = null;
      this.draggedOverListId = null;
      this.dragOverTarget = null;
    },
    
    onCardDrop(event, targetListId) {
      event.preventDefault();
      
      if (!this.draggedCard) return;
      
      this.moveCardToList(this.draggedCard, this.draggedCardSourceList, targetListId);
      
      this.draggedCard = null;
      this.draggedCardSourceList = null;
      this.draggedCardIndex = null;
      this.draggedOverListId = null;
      this.dragOverTarget = null;
    },
    
    reorderCardInSameList(card, fromIndex, toIndex) {
      if (fromIndex === toIndex) return;
      
      const listCards = this.main.cards
        .filter(c => c.idList === card.idList && !c.closed)
        .sort((a, b) => a.pos - b.pos);
      
      const [movedCard] = listCards.splice(fromIndex, 1);
      listCards.splice(toIndex, 0, movedCard);
      
      listCards.forEach((card, idx) => {
        card.pos = idx;
      });
      
      this.main.cards = [...this.main.cards];
      
      console.log(`Moved card from ${fromIndex} to ${toIndex}`);
      this.$emit('card-reordered', { card, fromIndex, toIndex });
    },
    
    moveCardToList(card, sourceListId, targetListId) {
      const actualCard = this.main.cards.find(c => c.id === card.id);
      if (actualCard) {
        actualCard.idList = targetListId;
        
        const targetCards = this.main.cards.filter(c => c.idList === targetListId && !c.closed);
        targetCards.forEach((card, idx) => {
          card.pos = idx;
        });
        
        const sourceCards = this.main.cards.filter(c => c.idList === sourceListId && !c.closed);
        sourceCards.forEach((card, idx) => {
          card.pos = idx;
        });
        
        this.main.cards = [...this.main.cards];
      }
      
      this.$emit('card-moved', { card, sourceListId, targetListId });
    },
    
    moveCardToDifferentList(card, sourceListId, targetListId, targetIndex) {
      const actualCard = this.main.cards.find(c => c.id === card.id);
      if (!actualCard) return;
      
      actualCard.idList = targetListId;
      
      const targetCards = this.main.cards.filter(c => c.idList === targetListId && !c.closed);
      
      let currentIndex = targetCards.findIndex(c => c.id === card.id);
      
      if (currentIndex !== -1) {
        targetCards.splice(currentIndex, 1);
      }
      
      let insertAt = Math.min(targetIndex, targetCards.length);
      targetCards.splice(insertAt, 0, actualCard);
      
      targetCards.forEach((card, idx) => {
        card.pos = idx;
      });
      
      const sourceCards = this.main.cards.filter(c => c.idList === sourceListId && !c.closed);
      sourceCards.forEach((card, idx) => {
        card.pos = idx;
      });
      
      this.main.cards = [...this.main.cards];
      
      this.$emit('card-moved', { card, sourceListId, targetListId, targetIndex });
    },
    
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
    
    openCardPopup(cardId) {
      const card = this.main.cards.find(c => c.id === cardId);
      if (card) {
        this.selectedCard = card;
        this.showPopup = true;
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
    },
    
    getCardChecklists(checklistIds) {
      if (!checklistIds || checklistIds.length === 0 || !this.main.checklists) return [];
      return this.main.checklists.filter(checklist => 
        checklistIds.includes(checklist.id)
      );
    }
  }
};
</script>
