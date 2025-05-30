<script setup>
import { defineProps, ref, onActivated, defineEmits, watch, computed } from 'vue'

const emit = defineEmits(['update:sliders'])

const props = defineProps(['series', 'sliders', 'characteristics']) // Recevoir les séries et sliders via props
const similaritiesTable = ref([]) // Tableau des similarités
const comparisonResult = ref(null) // Résultat de la comparaison entre deux séries
let editFeature = ref(false) // État pour afficher/masquer la personnalisation

// Copie locale pour l'édition
const localSliders = ref({ ...props.sliders })

// Quand on ouvre la personnalisation, synchroniser la copie locale
function openEdit() {
  localSliders.value = { ...props.sliders }
  editFeature.value = true
}

// Émettre la mise à jour au parent
function validerChanges() {
  emit('update:sliders', localSliders.value)
  editFeature.value = false
  // Réinitialiser les résultats
  comparisonResult.value = null
  similaritiesTable.value = []
}
watch(
  () => props.sliders,
  () => {
    const selectedSeries = props.series.filter(serie => serie.checked)
    if (typeAffichage.value === 1 && selectedSeries.length > 0) {
      calculerSimilaritesPourUneSerie(selectedSeries[0].name)
    } else if (typeAffichage.value === 2 && selectedSeries.length > 1) {
      calculerSimilaritesEntreDeuxSeries(selectedSeries[0].name, selectedSeries[1].name)
    }
  },
  { deep: true }
)

// Fonction pour calculer la similarité entre deux vecteurs
function cosineSimilarity(A, B) {
  let dotproduct = 0;
  let mA = 0;
  let mB = 0;
  for (let i = 0; i < A.length; i++) {
    dotproduct += (A[i] * B[i]);
    mA += (A[i] * A[i]);
    mB += (B[i] * B[i]);
  }
  mA = Math.sqrt(mA);
  mB = Math.sqrt(mB);
  const denom = mA * mB;
  if (!denom) return 0;
  const result = dotproduct / denom;
  return isNaN(result) ? 0 : result;
}

// Fonction pour récupérer les caractéristiques d'une série
function getFeatures(serieName, featureKeys) {
  const serie = props.characteristics.find(item => item.name === serieName)
  if (!serie) return featureKeys.map(() => [])

  // Récupérer les clés de colonnes du CSV
  const allKeys = Object.keys(serie)

  // Définir les plages d'index pour chaque feature
  const llamaSynopsisCols = allKeys.slice(1, 51)
  const audioCols = allKeys.slice(51, 56)
  const videoCols = allKeys.slice(56)

  // Pour chaque feature, retourne le vecteur pondéré par les sliders
  return featureKeys.flatMap(key => {
    if (key === 'llama_Synopsis') {
      return llamaSynopsisCols.map(col => (parseFloat(serie[col]) || 0) * (parseFloat(props.sliders[col]) || 0))
    }
    if (key === 'audio') {
      return audioCols.map(col => (parseFloat(serie[col]) || 0) * (parseFloat(props.sliders[col]) || 0))
    }
    if (key === 'vidéo') {
      return videoCols.map(col => (parseFloat(serie[col]) || 0) * (parseFloat(props.sliders[col]) || 0))
    }
    return []
  })
}

// Fonction pour calculer les similarités pour une série donnée
function calculerSimilaritesPourUneSerie(serie_name) {
  const selectedSerie = props.series.find(serie => serie.name === serie_name)
  if (!selectedSerie) return

  const featureKeys = ['llama_Synopsis', 'audio', 'vidéo'] // Clés des caractéristiques utilisées pour le calcul

  similaritiesTable.value = props.series
    .filter(serie => serie.name !== serie_name)
    .map(serie => {
      const featureSimilarities = featureKeys.map(key => {
        const weight = parseFloat(props.sliders[key]) || 0
        if (weight === 0) return null // Ignorer la feature si poids 0
        const featureVectorA = getFeatures(selectedSerie.name, [key])
        const featureVectorB = getFeatures(serie.name, [key])
        const similarity = cosineSimilarity(featureVectorA, featureVectorB)
        return {
          key,
          similarity,
          weight
        }
      }).filter(Boolean) // Retirer les null

      const weightedSimilarity = featureSimilarities.length > 0
        ? featureSimilarities.reduce(
            (sum, feature) => sum + feature.similarity * feature.weight,
            0
          ) / featureSimilarities.reduce((sum, feature) => sum + feature.weight, 0)
        : 0

      return {
        name: serie.name,
        similarity: weightedSimilarity,
        featureSimilarities: featureSimilarities,
        image: serie.image,
        description: serie.description
      }
    })
    

  similaritiesTable.value.sort((a, b) => b.similarity - a.similarity) // Trier par similarité décroissante
}

// Fonction pour calculer la similarité entre deux séries
function calculerSimilaritesEntreDeuxSeries(serie1Name, serie2Name) {
  const featureKeys = ['llama_Synopsis', 'audio', 'vidéo'] // Clés des caractéristiques utilisées pour le calcul

  const featureSimilarities = featureKeys.map(key => {
    const weight = parseFloat(props.sliders[key]) || 0
    if (weight === 0) return null
    const featureVectorA = getFeatures(serie1Name, [key])
    const featureVectorB = getFeatures(serie2Name, [key])
    const similarity = cosineSimilarity(featureVectorA, featureVectorB)
    return {
      key,
      similarity,
      weight
    }
  }).filter(Boolean)

  const weightedSimilarity = featureSimilarities.length > 0
    ? featureSimilarities.reduce(
        (sum, feature) => sum + feature.similarity * feature.weight,
        0
      ) / featureSimilarities.reduce((sum, feature) => sum + feature.weight, 0)
    : 0

  // Stocker le résultat dans comparisonResult
  comparisonResult.value = {
    serie1Name,
    serie2Name,
    weightedSimilarity,
    featureSimilarities
  }
}

// Charger les données et effectuer les calculs à chaque fois que le composant est activé
let typeAffichage = ref(0) // 1 pour une série, 2 pour deux séries
onActivated(async () => {
  try {
    console.log('Caractéristiques disponibles :', props.characteristics)
    // Vérifier si la liste des séries sélectionnées est vide
    const selectedSeries = props.series.filter(serie => serie.checked)
    if (selectedSeries.length === 0) {
      // Rediriger vers Home si aucune série n'est sélectionnée
      window.location.hash = '#/'
      return
    }
    console.log('Séries sélectionnées :', selectedSeries)
    console.log(selectedSeries.length)
    if (selectedSeries.length === 1) {
      typeAffichage.value = 1
      // Calculer les similarités pour la première série cochée
      calculerSimilaritesPourUneSerie(selectedSeries[0].name)
      
    } else if (selectedSeries.length === 2) {
      // Calculer les similarités entre deux séries cochées
      calculerSimilaritesEntreDeuxSeries(selectedSeries[0].name, selectedSeries[1].name)
      typeAffichage.value = 2
    }
    
    /* teste pour faire un calcul sur toutes les séries cochées
    else {
      // Calculer les similarités pour toutes les séries cochées
      props.series.forEach(serie => {
        if (serie.checked) {
          calculerSimilaritesPourUneSerie(serie.name)
        }
      })
    }
    */
  } catch (error) {
    console.error('Erreur lors du chargement des caractéristiques :', error)
  }
})

// Fonction pour afficher la similarité pour chaque feature
function showFeatureSimilarities(featureSimilarities) {
  const message = featureSimilarities
    .map(feature => `${feature.key} : ${(feature.similarity * 100).toFixed(2)}% (Pondération : ${feature.weight})`)
    .join('\n')
  alert(`Similarité par feature :\n${message}`)
}


const allKeys = computed(() => {
  // Prend la première série pour obtenir les colonnes
  const firstSerie = props.characteristics[0]
  return firstSerie ? Object.keys(firstSerie) : []
})

const llamaSynopsisCols = computed(() => allKeys.value.slice(1, 51))
const audioCols = computed(() => allKeys.value.slice(51, 56))
const videoCols = computed(() => allKeys.value.slice(56))

// Fonction utilitaire pour diviser un tableau en morceaux
function chunkArray(array, size) {
  const result = []
  for (let i = 0; i < array.length; i += size) {
    result.push(array.slice(i, i + size))
  }
  return result
}

const audioColsChunks = computed(() => chunkArray(audioCols.value, 10))
const videoColsChunks = computed(() => chunkArray(videoCols.value, 10))
const llamaSynopsisColsChunks = computed(() => chunkArray(llamaSynopsisCols.value, 10))

function syncCheckboxGroup(mainKey) {
  let group = []
  if (mainKey === 'llama_Synopsis') group = llamaSynopsisCols.value
  if (mainKey === 'audio') group = audioCols.value
  if (mainKey === 'vidéo') group = videoCols.value
  const value = localSliders.value[mainKey]
  group.forEach(key => {
    localSliders.value[key] = value
  })
}
</script>

<template>
  <div class="contenu">
    <h2>Résultats des séries sélectionnées</h2>
    <div id="personnalisation">
      <button @click="openEdit" v-if="!editFeature">personnaliser le resultat</button>
      <div v-if="editFeature">
        <button @click="validerChanges">Valider les changements</button>
        <h3>Personnalisation des critères</h3>
        <!-- Ligne dédiée pour les 3 critères principaux -->
        <div class="main-checkbox-row">
          <label v-for="key in ['llama_Synopsis', 'audio', 'vidéo']" :key="key" class="checkbox-item">
            <input
              type="checkbox"
              v-model="localSliders[key]"
              true-value="1"
              false-value="0"
              @change="syncCheckboxGroup(key)"
            />
            {{ key }}
          </label>
        </div>
        <!-- Grille pour les autres critères -->
        <div class="checkbox-grid-multi">
          <div>
            <strong>Audio</strong>
            <div class="checkbox-col" v-for="(chunk, idx) in audioColsChunks" :key="'audio-col-' + idx">
              <label
                v-for="key in chunk"
                :key="key"
                class="checkbox-item"
              >
                <input
                  type="checkbox"
                  v-model="localSliders[key]"
                  true-value="1"
                  false-value="0"
                />
                {{ key }}
              </label>
            </div>
          </div>
          <div>
            <strong>Vidéo</strong>
            <div class="checkbox-col" v-for="(chunk, idx) in videoColsChunks" :key="'video-col-' + idx">
              <label
                v-for="key in chunk"
                :key="key"
                class="checkbox-item"
              >
                <input
                  type="checkbox"
                  v-model="localSliders[key]"
                  true-value="1"
                  false-value="0"
                />
                {{ key }}
              </label>
            </div>
          </div>
          <div>
            <strong>llama_Synopsis</strong>
            <div class="checkbox-col" v-for="(chunk, idx) in llamaSynopsisColsChunks" :key="'llama-col-' + idx">
              <label
                v-for="key in chunk"
                :key="key"
                class="checkbox-item"
              >
                <input
                  type="checkbox"
                  v-model="localSliders[key]"
                  true-value="1"
                  false-value="0"
                />
                {{ key }}
              </label>
            </div>
          </div>
        </div>
        <button @click="validerChanges">Valider les changements</button>
        <button @click="editFeature = false">Annuler</button>
      </div>
    </div>
    <table v-if="typeAffichage===1" class="similarities-table">
      <caption>Tableau des similarités</caption>
      <thead>
        <tr>
          <th>Série</th>
          <th>Image</th>
          <th>Description</th>
          <th>Score de Similarité</th>
          <th>Action</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="item in similaritiesTable" :key="item.name">
          <td>{{ item.name }}</td>
          <td>
            <img :src="item.image" alt="Image de la série" class="serie-image" />
          </td>
          <td>{{ item.description }}</td>
          <td>{{ (item.similarity * 100).toFixed(2) }}%</td>
          <td>
            <button @click="showFeatureSimilarities(item.featureSimilarities)">Voir Similarité par Feature</button>
          </td>
        </tr>
      </tbody>
    </table>

    <!-- Affichage des résultats de la comparaison -->
    <div v-else-if="typeAffichage === 2 && comparisonResult" class="comparison-result">
      <h3>Comparaison entre {{ comparisonResult.serie1Name }} et {{ comparisonResult.serie2Name }}</h3>
      <p><strong>Similarité globale pondérée :</strong> {{ (comparisonResult.weightedSimilarity * 100).toFixed(2) }}%</p>
      <ul>
        <li v-for="feature in comparisonResult.featureSimilarities || []" :key="feature.key">
          <strong>{{ feature.key }} :</strong> {{ (feature.similarity * 100).toFixed(2) }}% (Pondération : {{ feature.weight }})
        </li>
      </ul>
    </div>
    <p v-else>Aucune série trouvée.</p>
  </div>
</template>

<style>
/* Styles pour le tableau */
table {
  width: 100%;
  border-collapse: collapse;
}

tbody {
  text-align: center;
}

td {
  padding: 10px;
  vertical-align: top;
}

.serie-image {
  max-width: 100px;
  max-height: 150px;
}

/* Styles pour les boutons */
button {
  padding: 5px 10px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

button:hover {
  background-color: #0056b3;
}

/* Styles pour la section de comparaison */
.comparison-result {
  margin-top: 20px;
  padding: 15px;
  border: 1px solid var(--border-color, #ddd);
  border-radius: 5px;
  background-color: var(--background-color, #f9f9f9);
  color: var(--text-color, #000);
}

/* Titre de la section */
.comparison-result h3 {
  margin-bottom: 10px;
}

/* Liste des similarités */
.comparison-result ul {
  list-style-type: none;
  padding: 0;
}

.comparison-result li {
  margin-bottom: 5px;
}

/* Mode sombre */
html.dark .comparison-result {
  --background-color: #333;
  --text-color: #00c7ec;
  --border-color: #555;
}

/* Styles pour la personnalisation */
#personnalisation {
  margin-bottom: 20px;
}

h3 {
  margin-top: 0;
}

/* Ligne principale des checkboxes */
.main-checkbox-row {
  display: flex;
  gap: 2em;
  justify-content: center;
  margin-bottom: 1em;
}

/* Grille des checkboxes */
.checkbox-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
  gap: 1em 2em;
  margin-bottom: 1em;
  justify-items: start;
}

/* Grille des checkboxes (multi-colonnes) */
.checkbox-grid-multi {
  display: flex;
  gap: 2em;
  justify-content: center;
  margin-bottom: 1em;
}

.checkbox-grid-multi > div {
  display: flex;
  flex-direction: column;
  gap: 0.5em;
  min-width: 180px;
}

.checkbox-grid-multi strong {
  margin-bottom: 0.5em;
  text-align: center;
  display: block;
}

/* Éléments individuels de la checkbox */
.checkbox-item {
  display: flex;
  align-items: center;
  gap: 0.5em;
  font-size: 1em;
}

.checkbox-col {
  display: flex;
  flex-direction: column;
  margin-bottom: 1em;
}
</style>