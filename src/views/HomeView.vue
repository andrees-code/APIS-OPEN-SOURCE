<template>
  <div class="api-vault">
    <header class="header">
      <h1><span class="icon">⚡</span> Free API <span class="highlight">Vault</span></h1>
      <p>Directorio curado para desarrolladores. Sin CORS, sin tarjetas, listas para usar.</p>
      
      <div class="controls">
        <div class="search-wrapper">
          <span class="search-icon">🔍</span>
          <input 
            v-model="searchQuery" 
            type="text" 
            class="search-bar" 
            placeholder="Buscar por nombre, descripción o tecnología..."
          />
        </div>
      </div>

      <nav class="filters primary-filters" aria-label="Filtros principales">
        <button 
          class="filter-btn" 
          :class="{ active: selectedParent === 'Todas' }"
          @click="selectParent('Todas')"
        >
          🌐 Todas
        </button>
        <button 
          v-for="parent in parentCategories" 
          :key="parent"
          class="filter-btn"
          :class="{ active: selectedParent === parent }"
          @click="selectParent(parent)"
        >
          {{ parent }}
        </button>
      </nav>

      <Transition name="slide-down">
        <nav v-if="selectedParent !== 'Todas'" class="filters secondary-filters" aria-label="Subfiltros">
          <button 
            class="sub-filter-btn" 
            :class="{ active: selectedChild === 'Todas' }"
            @click="selectedChild = 'Todas'"
          >
            ↳ Todo en {{ selectedParent }}
          </button>
          <button 
            v-for="child in subCategories" 
            :key="child"
            class="sub-filter-btn"
            :class="{ active: selectedChild === child }"
            @click="selectedChild = child"
          >
            {{ child }}
          </button>
        </nav>
      </Transition>
    </header>

    <div v-if="filteredApis.length === 0" class="empty-state">
      <span class="empty-icon">🛸</span>
      <h3>Ups, no encontramos ninguna API</h3>
      <p>Prueba buscando con otros términos o cambia la categoría seleccionada.</p>
      <button class="reset-btn" @click="resetFilters">Limpiar filtros</button>
    </div>

    <main v-else class="grid">
      <article 
        v-for="api in filteredApis" 
        :key="api.name" 
        class="api-card"
        @click="openModal(api)"
        tabindex="0"
        @keyup.enter="openModal(api)"
      >
        <div class="card-header">
          <span class="badge breadcrumb">
            {{ api.category }} <span class="divider">/</span> {{ api.subcategory }}
          </span>
          <span class="badge auth" :class="api.auth === 'No' ? 'auth-no' : 'auth-yes'">
            {{ api.auth === 'No' ? '🔓 Sin Auth' : '🔑 API Key' }}
          </span>
        </div>
        <h3>{{ api.name }}</h3>
        <p>{{ api.desc }}</p>
        <div class="card-footer">
          <span class="action-text">Ver documentación →</span>
        </div>
      </article>
    </main>

    <Transition name="fade">
      <div v-if="selectedApi" class="modal-backdrop" @click.self="closeModal" role="dialog" aria-modal="true">
        <div class="modal-content">
          <button class="close-btn" @click="closeModal" aria-label="Cerrar modal">×</button>
          
          <header class="modal-header">
            <div class="header-badges">
              <span class="badge breadcrumb">
                {{ selectedApi.category }} <span class="divider">/</span> {{ selectedApi.subcategory }}
              </span>
              <span class="badge auth" :class="selectedApi.auth === 'No' ? 'auth-no' : 'auth-yes'">
                {{ selectedApi.auth === 'No' ? '🔓 Sin Auth' : '🔑 Requiere API Key' }}
              </span>
            </div>
            <h2>{{ selectedApi.name }}</h2>
            <p>{{ selectedApi.desc }}</p>
            <a :href="selectedApi.docs" target="_blank" rel="noopener noreferrer" class="docs-btn">
              Documentación Oficial ↗
            </a>
          </header>

          <section class="modal-body">
            <div class="content-section">
              <h4>📍 Endpoint (GET)</h4>
              <div class="code-box">
                <code>{{ selectedApi.endpoint }}</code>
                <button class="copy-btn" @click="copyText(selectedApi.endpoint, 'Endpoint')">Copiar</button>
              </div>
            </div>

            <div class="content-section">
              <h4>🚀 Ejemplo de Implementación (Fetch API)</h4>
              <div class="code-box">
                <pre><code>{{ generateFetch(selectedApi.endpoint) }}</code></pre>
                <button class="copy-btn" @click="copyText(generateFetch(selectedApi.endpoint), 'Código')">Copiar</button>
              </div>
            </div>

            <div class="content-section">
              <h4>📦 Respuesta Esperada (JSON)</h4>
              <div class="code-box json">
                <pre><code>{{ JSON.stringify(selectedApi.ex, null, 2) }}</code></pre>
                <button class="copy-btn" @click="copyText(JSON.stringify(selectedApi.ex, null, 2), 'JSON')">Copiar</button>
              </div>
            </div>
          </section>
        </div>
      </div>
    </Transition>

    <Transition name="slide-up">
      <div v-if="toast.show" class="toast">
        ✅ {{ toast.message }} copiado al portapapeles
      </div>
    </Transition>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { apiList } from '@/data/apis.js'

// ESTADOS GLOBALES
const searchQuery = ref('')
const selectedParent = ref('Todas')
const selectedChild = ref('Todas')
const selectedApi = ref(null)

const toast = ref({ show: false, message: '' })

// COMPUTADAS PARA LOS FILTROS
const parentCategories = computed(() => {
  return [...new Set(apiList.map(api => api.category))].sort()
})

const subCategories = computed(() => {
  if (selectedParent.value === 'Todas') return []
  const apisInParent = apiList.filter(api => api.category === selectedParent.value)
  return [...new Set(apisInParent.map(api => api.subcategory))].sort()
})

const filteredApis = computed(() => {
  return apiList.filter(api => {
    const matchParent = selectedParent.value === 'Todas' || api.category === selectedParent.value
    const matchChild = selectedChild.value === 'Todas' || api.subcategory === selectedChild.value
    const matchSearch = api.name.toLowerCase().includes(searchQuery.value.toLowerCase()) || 
                        api.desc.toLowerCase().includes(searchQuery.value.toLowerCase())
    return matchParent && matchChild && matchSearch
  })
})

// MÉTODOS
const selectParent = (parent) => {
  selectedParent.value = parent
  selectedChild.value = 'Todas'
  searchQuery.value = ''
}

const resetFilters = () => {
  searchQuery.value = ''
  selectedParent.value = 'Todas'
  selectedChild.value = 'Todas'
}

const openModal = (api) => { 
  selectedApi.value = api
  document.body.style.overflow = 'hidden' 
}

const closeModal = () => { 
  selectedApi.value = null
  document.body.style.overflow = 'auto' 
}

const handleKeydown = (e) => {
  if (e.key === 'Escape' && selectedApi.value) closeModal()
}

onMounted(() => window.addEventListener('keydown', handleKeydown))
onUnmounted(() => window.removeEventListener('keydown', handleKeydown))

const copyText = async (text, type) => {
  try {
    await navigator.clipboard.writeText(text)
    toast.value = { show: true, message: type }
    setTimeout(() => { toast.value.show = false }, 3000)
  } catch (err) {
    console.error('Error al copiar: ', err)
  }
}

const generateFetch = (url) => {
  return `fetch('${url}')\n  .then(res => res.json())\n  .then(data => console.log(data))\n  .catch(err => console.error(err));`
}
</script>

<style lang="sass" scoped>
$bg-dark: #0f172a
$card-bg: #1e293b
$primary: #38bdf8
$primary-hover: #0284c7
$text-light: #f8fafc
$text-dim: #94a3b8

.api-vault
  background-color: $bg-dark
  min-height: 100vh
  color: $text-light
  padding: 4rem 1.5rem
  font-family: 'Inter', system-ui, sans-serif

.header
  max-width: 1200px
  margin: 0 auto 4rem
  text-align: center
  
  h1
    font-size: clamp(2.5rem, 5vw, 4rem)
    margin-bottom: 1rem
    font-weight: 800
    letter-spacing: -1px
    .highlight
      color: $primary
  
  p
    color: $text-dim
    font-size: 1.1rem
    margin-bottom: 2.5rem

.search-wrapper
  position: relative
  max-width: 600px
  margin: 0 auto 2rem
  
  .search-icon
    position: absolute
    left: 1.2rem
    top: 50%
    transform: translateY(-50%)
    font-size: 1.2rem
    opacity: 0.5
  
  .search-bar
    width: 100%
    padding: 1.2rem 1.5rem 1.2rem 3rem
    border-radius: 12px
    border: 2px solid #334155
    background: #0b1120
    color: white
    font-size: 1.1rem
    transition: all 0.3s ease
    &:focus
      outline: none
      border-color: $primary
      box-shadow: 0 0 0 4px rgba($primary, 0.1)

.filters
  display: flex
  flex-wrap: wrap
  justify-content: center
  gap: 0.8rem
  
  .filter-btn
    background: transparent
    border: 1px solid #334155
    color: $text-dim
    padding: 0.6rem 1.2rem
    border-radius: 8px
    cursor: pointer
    transition: all 0.2s ease
    font-weight: 600
    font-size: 0.9rem
    &:hover, &.active
      background: $primary
      border-color: $primary
      color: #0f172a

.primary-filters
  margin-bottom: 1rem

.secondary-filters
  background: rgba(#1e293b, 0.5)
  padding: 1rem
  border-radius: 12px
  border: 1px solid #334155
  margin-top: 1rem
  max-width: 800px
  margin-left: auto
  margin-right: auto

.sub-filter-btn
  background: transparent
  border: 1px dashed #475569
  color: #94a3b8
  padding: 0.4rem 1rem
  border-radius: 6px
  font-size: 0.85rem
  cursor: pointer
  transition: all 0.2s
  &:hover, &.active
    background: $primary
    color: #0f172a
    border-style: solid

.empty-state
  text-align: center
  padding: 5rem 1rem
  background: $card-bg
  border-radius: 16px
  max-width: 600px
  margin: 0 auto
  border: 1px dashed #334155
  
  .empty-icon
    font-size: 4rem
    display: block
    margin-bottom: 1rem
  h3
    font-size: 1.5rem
    margin-bottom: 0.5rem
  p
    color: $text-dim
    margin-bottom: 1.5rem
  .reset-btn
    background: $primary
    color: #000
    border: none
    padding: 0.8rem 1.5rem
    border-radius: 8px
    font-weight: bold
    cursor: pointer

.grid
  max-width: 1200px
  margin: 0 auto
  display: grid
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr))
  gap: 1.5rem

.api-card
  background: $card-bg
  border: 1px solid #334155
  border-radius: 16px
  padding: 1.8rem
  cursor: pointer
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1)
  display: flex
  flex-direction: column
  position: relative
  overflow: hidden
  
  &:hover, &:focus
    outline: none
    transform: translateY(-5px)
    border-color: $primary
    box-shadow: 0 10px 40px -10px rgba($primary, 0.3)
    
    .action-text
      transform: translateX(5px)

  h3
    margin: 1rem 0 0.5rem
    font-size: 1.4rem
    color: $text-light
  
  p
    color: $text-dim
    font-size: 0.95rem
    line-height: 1.6
    flex-grow: 1
    margin-bottom: 1.5rem

  .card-footer
    border-top: 1px solid #334155
    padding-top: 1rem
    margin-top: auto

  .action-text
    color: $primary
    font-weight: 600
    font-size: 0.9rem
    display: inline-block
    transition: transform 0.2s ease

.card-header, .header-badges
  display: flex
  justify-content: space-between
  align-items: center
  gap: 1rem
  margin-bottom: 1rem

.badge
  padding: 0.3rem 0.8rem
  border-radius: 6px
  font-size: 0.75rem
  font-weight: 800
  letter-spacing: 0.5px
  white-space: nowrap

.badge.breadcrumb
  background: rgba($primary, 0.15)
  color: $primary
  text-transform: uppercase

.badge.auth-no
  background: rgba(16, 185, 129, 0.1)
  color: #10b981
  border: 1px solid rgba(16, 185, 129, 0.3)

.badge.auth-yes
  background: rgba(245, 158, 11, 0.1)
  color: #f59e0b
  border: 1px solid rgba(245, 158, 11, 0.3)

.breadcrumb
  .divider
    color: #64748b
    margin: 0 4px

.modal-backdrop
  position: fixed
  inset: 0
  background: rgba(15, 23, 42, 0.8)
  backdrop-filter: blur(8px)
  display: flex
  align-items: center
  justify-content: center
  z-index: 100
  padding: 1rem

.modal-content
  background: #0f172a
  border: 1px solid #334155
  width: 100%
  max-width: 850px
  max-height: 90vh
  border-radius: 24px
  overflow-y: auto
  position: relative
  box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.7)

.close-btn
  position: sticky
  float: right
  top: 1.5rem
  right: 1.5rem
  background: #1e293b
  color: $text-dim
  border: 1px solid #334155
  width: 40px
  height: 40px
  border-radius: 50%
  font-size: 1.5rem
  display: flex
  align-items: center
  justify-content: center
  cursor: pointer
  transition: all 0.2s
  z-index: 10
  &:hover
    background: #ef4444
    color: white
    border-color: #ef4444

.modal-header
  padding: 3rem 3rem 2rem
  border-bottom: 1px solid #1e293b
  background: $card-bg
  
  h2
    font-size: 2.5rem
    margin: 1rem 0 0.5rem
    color: white
  p
    color: $text-dim
    font-size: 1.1rem
    margin-bottom: 1.5rem
    line-height: 1.6
  
  .docs-btn
    display: inline-flex
    align-items: center
    background: $primary
    color: #0f172a
    text-decoration: none
    padding: 0.8rem 1.5rem
    border-radius: 8px
    font-weight: 700
    transition: all 0.2s
    &:hover
      background: $text-light

.modal-body
  padding: 3rem
  
  .content-section
    margin-bottom: 2.5rem
    h4
      color: $text-light
      margin-bottom: 1rem
      font-size: 1.1rem
      font-weight: 600

.code-box
  background: #0b1120
  border: 1px solid #1e293b
  border-radius: 12px
  padding: 1.2rem
  display: flex
  justify-content: space-between
  align-items: flex-start
  gap: 1.5rem
  position: relative
  
  code, pre
    font-family: 'Fira Code', 'Cascadia Code', monospace
    color: #a5b4fc
    margin: 0
    font-size: 0.95rem
    line-height: 1.5
    overflow-x: auto
    white-space: pre-wrap
  
  &.json code
    color: #fde047

  .copy-btn
    flex-shrink: 0
    background: #1e293b
    color: $text-light
    border: 1px solid #334155
    padding: 0.5rem 1rem
    border-radius: 6px
    cursor: pointer
    font-size: 0.85rem
    font-weight: 600
    transition: all 0.2s
    &:hover
      background: $primary
      color: #0f172a
      border-color: $primary

.toast
  position: fixed
  bottom: 2rem
  right: 2rem
  background: #10b981
  color: #064e3b
  padding: 1rem 1.5rem
  border-radius: 8px
  font-weight: 700
  box-shadow: 0 10px 25px rgba(16, 185, 129, 0.3)
  z-index: 1000
  display: flex
  align-items: center
  gap: 0.5rem

/* ANIMACIONES */
.fade-enter-active, .fade-leave-active
  transition: opacity 0.3s ease
.fade-enter-from, .fade-leave-to
  opacity: 0

.slide-up-enter-active, .slide-up-leave-active
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1)
.slide-up-enter-from, .slide-up-leave-to
  opacity: 0
  transform: translateY(20px)

.slide-down-enter-active, .slide-down-leave-active
  transition: all 0.3s ease-out
.slide-down-enter-from, .slide-down-leave-to
  opacity: 0
  transform: translateY(-10px)
</style>