<script setup>
import { ref } from "vue";

const props = defineProps({
  imageMatrix: {
    type: Array,
    required: true,
  },
  conversionMatrix: {
    type: Array,
    required: true,
  },
});

const getColor = (value) => {
  if (value === -1) return "transparent"; // LED non présente
  return `rgb(${value})`; // Exemple de conversion en couleur
};
</script>

<template>
  <div class="matrix">
    <div
      v-for="(row, rowIndex) in props.conversionMatrix"
      :key="rowIndex"
      class="row"
    >
      <div v-for="(led, colIndex) in row" :key="colIndex">
        <div
          v-if="led !== -1"
          class="led"
          :style="{
            backgroundColor: getColor(props.imageMatrix[rowIndex][colIndex]),
            visibility: 'visible',
          }"
        ></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.matrix {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.row {
  display: flex;
}

.led {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  margin: 2px;
  visibility: hidden;
}

.hide {
  visibility: vhiddden;
}
</style>
