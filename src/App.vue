<script setup>
import { ref, watch } from "vue";
import * as ImageTracerModule from "imagetracerjs";

// Handle default exports from commonjs modules in Vite
const ImageTracer = ImageTracerModule.default || ImageTracerModule;

const imageUrl = ref(null);
const svgResultRaw = ref(null);
const svgResult = ref(null);
const isProcessing = ref(false);
const downloadUrl = ref(null);

const useSolidColor = ref(true);
const solidColor = ref('#000000');
const removeBackground = ref(true);

watch([useSolidColor, solidColor, removeBackground, svgResultRaw], () => {
  if (!svgResultRaw.value) return;
  
  let finalSvg = svgResultRaw.value;
  
  if (useSolidColor.value || removeBackground.value) {
    try {
      const parser = new DOMParser();
      const doc = parser.parseFromString(finalSvg, "image/svg+xml");
      const paths = doc.querySelectorAll("path");
      
      paths.forEach(p => {
        const fill = p.getAttribute("fill");
        
        // Remove white/light backgrounds
        if (removeBackground.value && fill && fill.startsWith("rgb")) {
          const match = fill.match(/\d+/g);
          if (match && match.length >= 3) {
            const r = parseInt(match[0]);
            const g = parseInt(match[1]);
            const b = parseInt(match[2]);
            if (r > 240 && g > 240 && b > 240) {
              p.remove();
              return;
            }
          }
        }
        
        // Enforce solid color on both fill and stroke
        if (useSolidColor.value) {
          if (p.hasAttribute("fill") && p.getAttribute("fill") !== "none") {
            p.setAttribute("fill", solidColor.value);
          }
          if (p.hasAttribute("stroke") && p.getAttribute("stroke") !== "none") {
            p.setAttribute("stroke", solidColor.value);
          }
        }
      });
      
      finalSvg = new XMLSerializer().serializeToString(doc);
    } catch(e) {
      console.error("Erro ao processar SVG", e);
    }
  }
  
  svgResult.value = finalSvg;
  
  // Revoke previous URL to avoid memory leaks
  if (downloadUrl.value) {
    URL.revokeObjectURL(downloadUrl.value);
  }
  const blob = new Blob([finalSvg], { type: "image/svg+xml;charset=utf-8" });
  downloadUrl.value = URL.createObjectURL(blob);
});

const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  const reader = new FileReader();
  reader.onload = (e) => {
    imageUrl.value = e.target.result;
    processImage(e.target.result);
  };
  reader.readAsDataURL(file);
};

const processImage = (imgDataUrl) => {
  isProcessing.value = true;
  svgResultRaw.value = null;
  svgResult.value = null;

  // See imagetracerjs options for more detail
  const options = {
    strokewidth: 1,
    linefilter: true,
    scale: 1,
  };

  // imageToSVG expects image URL, callback, options
  ImageTracer.imageToSVG(
    imgDataUrl,
    (svgStr) => {
      // Trigger the watcher which will apply colors and build the URL
      svgResultRaw.value = svgStr;
      
      isProcessing.value = false;
    },
    options,
  );
};
</script>

<template>
  <div class="app-container">
    <header>
      <h1>Gerador de SVG com ImageTracer.js</h1>
      <p>
        Faça upload de uma imagem (JPG/PNG) para transformá-la em um vetor SVG.
      </p>
    </header>

    <main>
      <div class="upload-area">
        <label class="upload-btn">
          Selecionar Imagem
          <input
            type="file"
            accept="image/png, image/jpeg, image/webp"
            @change="handleFileUpload"
            hidden
          />
        </label>
      </div>

      <div class="options-area" v-if="imageUrl">
        <label class="option-row">
          <input type="checkbox" v-model="removeBackground" />
          Remover Fundo Branco (Transparente)
        </label>
        <label class="option-row">
          <input type="checkbox" v-model="useSolidColor" />
          Preencher com Cor Sólida
        </label>
        <div class="color-picker-row" v-if="useSolidColor">
          <input type="color" v-model="solidColor" class="color-input" />
          <span class="color-value">{{ solidColor }}</span>
        </div>
      </div>

      <div class="workspace" v-if="imageUrl || isProcessing || svgResult">
        <div class="image-panel">
          <h2>Original</h2>
          <div class="preview-box">
            <img v-if="imageUrl" :src="imageUrl" alt="Imagem Original" />
            <div v-else class="placeholder">Aguardando imagem...</div>
          </div>
        </div>

        <div class="image-panel">
          <h2>Vetor SVG</h2>
          <div class="preview-box">
            <div v-if="isProcessing" class="loading-state">
              <span class="spinner"></span>
              Processando vetores...
            </div>
            <div
              v-else-if="svgResult"
              class="svg-wrapper"
              v-html="svgResult"
            ></div>
            <div v-else class="placeholder">Nenhum SVG gerado</div>
          </div>
        </div>
      </div>

      <div class="actions" v-if="svgResult">
        <a
          :href="downloadUrl"
          download="imagem-vetorizada.svg"
          class="download-btn"
        >
          Salvar SVG
        </a>
      </div>
    </main>
  </div>
</template>

<style scoped>
.app-container {
  max-width: 1000px;
  margin: 0 auto;
  padding: 2rem;
  font-family:
    system-ui,
    -apple-system,
    sans-serif;
  color: #e0e0e0;
}

header {
  text-align: center;
  margin-bottom: 3rem;
}

header h1 {
  background: -webkit-linear-gradient(45deg, #42d392, #647eff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  margin-bottom: 0.5rem;
}

.upload-area {
  display: flex;
  justify-content: center;
  margin-bottom: 2rem;
}

.upload-btn {
  background-color: #646cff;
  color: white;
  padding: 12px 28px;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 4px 12px rgba(100, 108, 255, 0.3);
}

.upload-btn:hover {
  background-color: #747bff;
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(100, 108, 255, 0.4);
}

.options-area {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  margin-bottom: 2rem;
  background: #1e1e1e;
  padding: 1.5rem;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  max-width: 400px;
  margin-left: auto;
  margin-right: auto;
}

.option-row {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 1.1rem;
  cursor: pointer;
  user-select: none;
}

.option-row input[type="checkbox"] {
  width: 20px;
  height: 20px;
  cursor: pointer;
}

.color-picker-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.color-input {
  width: 50px;
  height: 50px;
  padding: 0;
  border: none;
  border-radius: 8px;
  background: none;
  cursor: pointer;
}

.color-input::-webkit-color-swatch-wrapper {
  padding: 0;
}

.color-input::-webkit-color-swatch {
  border: 2px solid #444;
  border-radius: 8px;
}

.color-value {
  font-family: monospace;
  font-size: 1.1rem;
  color: #42d392;
}

.workspace {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2rem;
  margin-bottom: 2rem;
}

@media (max-width: 768px) {
  .workspace {
    grid-template-columns: 1fr;
  }
}

.image-panel {
  background: #1e1e1e;
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
}

.image-panel h2 {
  font-size: 1.1rem;
  margin-top: 0;
  margin-bottom: 1rem;
  color: #abb2bf;
  border-bottom: 1px solid #333;
  padding-bottom: 0.5rem;
}

.preview-box {
  min-height: 300px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #2a2a2a;
  border-radius: 8px;
  overflow: hidden;
  padding: 1rem;
}

.preview-box img {
  max-width: 100%;
  max-height: 400px;
  object-fit: contain;
}

.svg-wrapper {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}

.svg-wrapper :deep(svg) {
  max-width: 100%;
  max-height: 400px;
  width: auto;
  height: auto;
}

.placeholder {
  color: #666;
  font-style: italic;
}

.loading-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  color: #42d392;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid rgba(66, 211, 146, 0.2);
  border-top-color: #42d392;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.actions {
  display: flex;
  justify-content: center;
  margin-top: 3rem;
}

.download-btn {
  background-color: #42d392;
  color: #1a1a1a;
  padding: 14px 32px;
  border-radius: 8px;
  font-weight: bold;
  font-size: 1.1rem;
  text-decoration: none;
  transition: all 0.2s ease;
  box-shadow: 0 4px 12px rgba(66, 211, 146, 0.3);
}

.download-btn:hover {
  background-color: #33a873;
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(66, 211, 146, 0.4);
}
</style>
