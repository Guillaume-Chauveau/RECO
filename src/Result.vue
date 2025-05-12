<script setup>
import { ref, onMounted, watch } from 'vue'
import { selectedSeries, sliders } from './store.js'

const df = ref([]) // Tableau réactif pour stocker les données du CSV
const allSeries = ref([]) // Liste complète des séries avec leurs descriptions et images
const similaritiesTable = ref([]) // Tableau des similarités
const comparisonResult = ref(null) // Résultat de la comparaison

// Fonction pour charger la liste complète des séries
async function loadAllSeries() {
  try {
    const response = await fetch('/RECO/data/Series.json') // Chemin vers le fichier JSON contenant toutes les séries
    if (!response.ok) {
      throw new Error(`Erreur lors du chargement des séries : ${response.statusText}`)
    }
    allSeries.value = await response.json()
  } catch (error) {
    console.error('Erreur lors du chargement des séries :', error)
  }
}

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

// Fonction pour obtenir l'image d'une série
function getSerieImage(serieName) {
  const serie = allSeries.value.find(s => s.name === serieName)
  return serie && serie.image ? serie.image : '/path/to/default-image.png' // Image par défaut si aucune image n'est disponible
}

// Fonction pour afficher la description d'une série
function showDescription(serieName) {
  const serie = allSeries.value.find(s => s.name === serieName)
  if (serie && serie.description) {
    alert(`Description de "${serie.name}": ${serie.description}`)
  } else {
    alert("Description non disponible pour cette série.")
  }
}

// Fonction pour exécuter les calculs en fonction des séries sélectionnées
function executerCalculs() {
  if (selectedSeries.value.length === 1) {
    const serieName = selectedSeries.value[0].name
    calculerSimilaritesPourUneSerie(serieName)
  } else if (selectedSeries.value.length === 2) {
    const [serie1, serie2] = selectedSeries.value.map(serie => serie.name)
    comparerDeuxSeries(serie1, serie2)
  } else {
    similaritiesTable.value = []
    comparisonResult.value = null
  }
}

// Fonction pour calculer les similarités pour une série
function calculerSimilaritesPourUneSerie(serieName) {
  similaritiesTable.value = df.value.map(row => {
    const otherSerieName = row['name']
    if (otherSerieName === serieName) return null

    // Exemple de calcul de similarité (ajustez selon vos besoins)
    const similarity = Math.random() // Remplacez par un vrai calcul
    return {
      name: otherSerieName,
      similarity,
      details: {
        llama_Synopsis: similarity * 0.4,
        audio: similarity * 0.3,
        vidéo: similarity * 0.3
      }
    }
  }).filter(item => item !== null)
}

// Fonction pour comparer deux séries
function comparerDeuxSeries(serie1, serie2) {
  // Exemple de comparaison (ajustez selon vos besoins)
  comparisonResult.value = {
    serie1,
    serie2,
    similarity: Math.random(), // Remplacez par un vrai calcul
    details: {
      llama_Synopsis: Math.random(),
      audio: Math.random(),
      vidéo: Math.random()
    }
  }
}

// Charger les données CSV et la liste complète des séries au montage
onMounted(async () => {
  await loadAllSeries()
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
              <img :src="getSerieImage(serie.name)" alt="Image de la série" class="serie-image" />
              <p>{{ allSeries.value.find(s => s.name === serie.name)?.description || "Description non disponible pour cette série" }}</p>
            </div>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
  <p v-else>Aucune série sélectionnée.</p>

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