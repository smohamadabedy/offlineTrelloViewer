<template>
  
    <template v-if="!versionSmall">
      <div class="label-bar" 
  :style="{ 'background': backgroundColor, 'height': barHeight, 'width':'150px','display':'flex','border-radius':'0.2rem'}">
      <svg
      class="trello-sidebar-label-svg" 
      :data-label-key="labelKey" 
      :width="barHeight" 
      :height="barHeight" 
      viewBox="0 0 44 44"
      v-html="labelSVG"
    ></svg>
    <div v-if="!showNone">
      <small v-if="!showLabel" class="p-1">{{ labelKey }}</small>
      <small v-if="showLabel" class="p-1">{{ displayValue }}</small>
    </div>
  </div>

    </template>
    <template v-else=>
      <svg
      class="trello-sidebar-label-svg" 
      :data-label-key="labelKey" 
      :width="15" 
      :height="15" 
      viewBox="0 0 44 44"
      v-html="labelSVG"
    ></svg>
    </template>
</template>

<script>
export default {
  name: 'LabelBar',
  props: {
    labelKey: {
      type: String,
      required: true,
      default: ''
    },
    value: {
      type: String,
      default: ''
    },
    displayValue: {
      type: String,
      default: ''
    },
    showNone:{
      type: Boolean,
      default: false,
    },
    showLabel:{
      type: Boolean,
      default: false,
    },
    versionSmall:{
      type: Boolean,
      default: false,
    },
    labelColorMap: {
      type: Object,
      required: true,
      default: () => ({})
    }
  },
  computed: {
    backgroundColor() {
      if (!this.labelKey) return '#888';
      return this.labelColorMap[this.labelKey] || '#888';
    },
    labelSVG() {
      if (!this.labelKey) return '';
      return this.generateLabelSVG(this.labelKey);
    },
    barHeight(){
      return !this.showNone ? '32px' : '22px';
    }
  },
  methods: {
    generateLabelSVG(labelKey) {
      if (!labelKey) return '';
      
      const color = this.labelColorMap[labelKey] || '#888';
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
      
      const pattern = patternMap[labelKey] || 'stripes';
      const darkerColor = this.adjustColorBrightness(color, -50);
      const lighterColor = this.adjustColorBrightness(color, 20);
      
      // Only use alphanumeric characters and underscores
      const safeKey = labelKey.replace(/[^a-zA-Z0-9]/g, '_');
      const patternId = `pat-${safeKey}`;
      const clipId = `clip-${safeKey}`;
      const gradientId = `grad-${safeKey}`;
      
      const rectPath = `M ${rectX + radius} ${rectY} h ${rectW - radius * 2} a ${radius} ${radius} 0 0 1 ${radius} ${radius} v ${rectH - radius * 2} a ${radius} ${radius} 0 0 1 -${radius} ${radius} h -${rectW - radius * 2} a ${radius} ${radius} 0 0 1 -${radius} -${radius} v -${rectH - radius * 2} a ${radius} ${radius} 0 0 1 ${radius} -${radius} z`;
      
      let patternSVG = '';
      
      switch(pattern) {
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
      if (!hex) return 'rgb(136,136,136)';
      
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
    }
  }
};
</script>

<style scoped>

.trello-sidebar-label-svg {
  display: block;
  border-radius: 3px;
  flex-shrink: 0;
}

small {
  color: white;
  text-shadow: 0 1px 2px rgba(0,0,0,0.3);
  font-weight: 500;
}
</style>