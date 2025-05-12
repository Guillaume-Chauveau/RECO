<script setup>
import { ref, onMounted, watch } from 'vue'
import { selectedSeries, sliders } from './store.js'

const df = ref([]) // Tableau réactif pour stocker les données du CSV
const similaritiesTable = ref([]) // Tableau des similarités
const comparisonResult = ref(null) // Résultat de la comparaison

const features = ['llama_Synopsis', 'audio', 'vidéo']

// Fonction pour charger le fichier CSV
async function loadCSV(url) {
  try {
    const response = await fetch(url)
    if (!response.ok) {
      throw new Error(`Erreur lors du chargement du fichier CSV : ${response.statusText}`)
    }
    const text = await response.text()
    const rows = text.split('\n').map(row => row.split(','))
    const headers = rows[0]
    const data = rows.slice(1).map(row => {
      const obj = {}
      headers.forEach((header, index) => {
        obj[header] = row[index]
      })
      return obj
    })
    df.value = data
  } catch (error) {
    console.error('Erreur lors du chargement du CSV :', error)
  }
}

// Fonction pour extraire les caractéristiques d'une série
function get_features(serie_name, features, df) {
  const feats = []
  const feature_names = {
    'llama_Synopsis': [1, 51],
    'audio': [51, 56],
    'vidéo': [56, Object.keys(df.value[0]).length]
  }

  if (features in feature_names) {
    const df_serie = df.value.filter(row => row['name'] === serie_name)
    if (df_serie.length > 0) {
      const idx = feature_names[features]
      const keys = Object.keys(df_serie[0]).slice(idx[0], idx[1])
      feats.push(...keys.map(key => parseFloat(df_serie[0][key]) || 0))
    }
  } else {
    console.error("Nom de caractéristique inconnu")
  }

  return feats
}

// Fonction pour calculer la similarité cosinus
function cosine_similarity(vecA, vecB) {
  const dotProduct = vecA.reduce((sum, value, index) => sum + value * vecB[index], 0)
  const magnitudeA = Math.sqrt(vecA.reduce((sum, value) => sum + value * value, 0))
  const magnitudeB = Math.sqrt(vecB.reduce((sum, value) => sum + value * value, 0))
  return dotProduct / (magnitudeA * magnitudeB)
}

// Fonction pour calculer les similarités avec toutes les séries
function calculerSimilaritesPourUneSerie(serie_name) {
  similaritiesTable.value = df.value.map(row => {
    const otherSerieName = row['name']
    if (otherSerieName === serie_name) return null

    const weightedSimilarities = features.map(feature => {
      const features_1 = get_features(serie_name, feature, df)
      const features_2 = get_features(otherSerieName, feature, df)
      const sliderValue = Number(sliders.value[feature]) || 1
      if (sliderValue === 0) return { similarity: 0, weight: 0 }
      const similarityScore = cosine_similarity(features_1, features_2) * sliderValue
      return { similarity: similarityScore, weight: sliderValue }
    })

    const totalWeight = weightedSimilarities.reduce((sum, item) => sum + item.weight, 0)
    const averageSimilarity = totalWeight > 0
      ? weightedSimilarities.reduce((sum, item) => sum + item.similarity, 0) / totalWeight
      : 0

    return { 
      name: otherSerieName, 
      similarity: averageSimilarity, 
      details: {
        llama_Synopsis: weightedSimilarities[0].similarity,
        audio: weightedSimilarities[1].similarity,
        vidéo: weightedSimilarities[2].similarity
      }
    }
  }).filter(item => item !== null)

  similaritiesTable.value.sort((a, b) => b.similarity - a.similarity)
}

// Fonction pour comparer deux séries
function comparerDeuxSeries(serie1, serie2) {
  const similarities = features.map(feature => {
    const features_1 = get_features(serie1, feature, df)
    const features_2 = get_features(serie2, feature, df)
    return cosine_similarity(features_1, features_2)
  })

  const averageSimilarity = similarities.reduce((sum, val) => sum + val, 0) / similarities.length
  comparisonResult.value = { 
    serie1, 
    serie2, 
    similarity: averageSimilarity,
    details: {
      llama_Synopsis: similarities[0],
      audio: similarities[1],
      vidéo: similarities[2]
    }
  }
}

// Fonction pour exécuter les calculs en fonction des séries sélectionnées
function executerCalculs() {
  if (selectedSeries.value.length === 1) {
    const serieName = selectedSeries.value[0]['name']
    calculerSimilaritesPourUneSerie(serieName)
  } else if (selectedSeries.value.length === 2) {
    const [serie1, serie2] = selectedSeries.value.map(serie => serie['name'])
    comparerDeuxSeries(serie1, serie2)
  }
}

// Fonction pour obtenir l'image d'une série
function getSerieImage(serieName) {
  const serie = selectedSeries.value.find(s => s.name === serieName)
  return serie ? serie.image : null
}

// Fonction pour afficher la description d'une série
function showDescription(serieName) {
  const serie = selectedSeries.value.find(s => s.name === serieName)
  if (serie && serie.description) {
    alert(`Description de "${serie.name}": ${serie.description}`)
  } else {
    alert("Description non disponible pour cette série.")
  }
}

// Charger les données CSV et exécuter les calculs au montage
onMounted(async () => {
  await loadCSV('/RECO/data/characteristics.csv')
  executerCalculs()
})

// Recalculer les résultats si les séries sélectionnées changent
watch(selectedSeries, () => {
  executerCalculs()
})
</script>

<template>
  <h2>Résultats des séries sélectionnées</h2>
  
  <!-- Affichage des séries sélectionnées -->
  <div v-if="selectedSeries.length > 0">
    <table>
      <tbody>
        <tr v-for="serie in selectedSeries" :key="serie.id">
          <td class="checkbox-wrapper-50">
            <div class="cell-content">
              <p class="serie-title">{{ serie.name }}</p>
              <img :src="serie.image" alt="Image de la série" v-if="serie.image" class="serie-image" />
              <p v-if="serie.description">{{ serie.description }}</p>
            </div>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
  <p v-else>Aucune série sélectionnée.</p>

  <h3>Valeurs des sliders :</h3>
  <ul>
    <li>Vidéo : {{ sliders.vidéo }}</li>
    <li>Scénario : {{ sliders.llama_Synopsis }}</li>
    <li>Audio : {{ sliders.audio }}</li>
  </ul>

  <h3 v-if="selectedSeries.length === 1">Tableau des similarités</h3>
  <table v-if="similaritiesTable.length > 0 && selectedSeries.length === 1">
    <thead>
      <tr>
        <th>Image</th>
        <th>Série</th>
        <th>Score de Similarité (%)</th>
        <th>llama_Synopsis (%)</th>
        <th>Audio (%)</th>
        <th>Vidéo (%)</th>
        <th>Description</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="item in similaritiesTable" :key="item.name">
        <td>
          <img 
            :src="getSerieImage(item.name)" 
            alt="Image de la série" 
            class="serie-image" 
            v-if="getSerieImage(item.name)" 
          />
        </td>
        <td>{{ item.name }}</td>
        <td>{{ (item.similarity * 100).toFixed(2) }}</td>
        <td>{{ (item.details.llama_Synopsis * 100).toFixed(2) }}</td>
        <td>{{ (item.details.audio * 100).toFixed(2) }}</td>
        <td>{{ (item.details.vidéo * 100).toFixed(2) }}</td>
        <td>
          <button @click="showDescription(item.name)">Voir la description</button>
        </td>
      </tr>
    </tbody>
  </table>

  <h3 v-if="selectedSeries.length === 2">Comparaison entre deux séries</h3>
  <p v-if="comparisonResult">
    Similarité entre {{ comparisonResult.serie1 }} et {{ comparisonResult.serie2 }} :
    {{ (comparisonResult.similarity * 100).toFixed(2) }}%
  </p>
  <ul v-if="comparisonResult">
    <li>llama_Synopsis : {{ (comparisonResult.details.llama_Synopsis * 100).toFixed(2) }}%</li>
    <li>Audio : {{ (comparisonResult.details.audio * 100).toFixed(2) }}%</li>
    <li>Vidéo : {{ (comparisonResult.details.vidéo * 100).toFixed(2) }}%</li>
  </ul>
</template>

<style>
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

.cell-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.serie-title {
  font-size: 1.2em;
  font-weight: bold;
  margin-bottom: 10px;
}

.serie-image {
  max-width: 100px;
  max-height: 150px;
  margin-bottom: 10px;
}

button {
  padding: 5px 10px;
  font-size: 0.9em;
  cursor: pointer;
}
</style>